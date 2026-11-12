---
tags: [nanza-api, infra, plan]
status: plan — 2026-11-09 (Skylar, voice); after the release, nothing built; infra only, no app change
---

# S3 backup for user-generated images

**Goal:** if user-generated images in the media bucket are deleted or overwritten, we can
get them back. That covers our own bugs, a fat-finger mistake in the console, and a
compromised account. Entity (catalogue) art is **out of scope**: it can be re-pulled from
the card sources and re-run through the migration images stage.

**No app change.** Uploads keep the same keys in the same bucket. Everything here is a
bucket setting or a second bucket, so it can ship any day, independent of the app release.
Done by Skylar in the console (Claude never runs AWS commands).

## What gets protected

Every prefix the API writes (`uploadImage(file, '<prefix>')` → `<prefix>/<id>.<ext>`), as of
2026-11-09:

| Prefix | Written by | Back up? |
|---|---|---|
| `post/` | post images + gallery photos (incl. GIFs as animated WebP) | **yes** |
| `profile/` | avatars | **yes** |
| `listing/` | listing photos + scan items (`scanItem.ts` writes here too) | **yes** |
| `bulk/` | bulk lot photos | **yes** |
| `list/` | collection / binder banners | **yes** |
| `group-banner/`, `group-thumbnail/` | group images | **yes** |
| `id-verifications/` | seller ID uploads (private, `uploadPrivateImage`) | **yes**: sensitive, keep the replica private too |
| `brand/`, `set/`, `supported-tag-value/`, `social-link-type/` | admin-curated art | **yes**: small, and not re-pullable |
| `entity/` | catalogue card art | no: re-pullable |
| OG share images, transcribe audio | `services/og`, `services/transcribe` | no: regenerated / temporary |

Re-check this list whenever a new upload prefix is added.

## Where things live
- **This plan:** `nanza-vault/nanza-api/solution-designs/s3-ugc-backup.md`.
- **Originals:** stay where they are, in the media bucket, under the same folders (`post/`,
  `profile/`, …).
- **Backups:** a **new bucket in a new backup AWS account**. Each image keeps the **same
  folder and file name** as the original (e.g. `post/<id>.webp`), so restoring is a copy
  back to the same path.

## Step by step

### Part A: versioning (any day, ~5 min)
1. S3 console → open the **media bucket** → **Properties** tab.
2. **Bucket Versioning** → Edit → **Enable** → Save.
3. **Management** tab → **Create lifecycle rule**:
   - Name: `expire-old-versions`, scope **apply to all objects**.
   - Tick **Permanently delete noncurrent versions of objects** → **60** days.
   - Tick **Delete expired object delete markers or incomplete multipart uploads**,
     incomplete uploads after **7** days.
   - Create rule.
4. Check it works: upload a test file, delete it, turn on **Show versions**, and the file
   is still there under a delete marker. Delete the marker to bring it back.

From here on, every delete or overwrite can be undone for 60 days. Our code already
deletes the old object whenever an image is replaced, which is why the lifecycle rule is
needed to keep storage from growing.

### Part B: backup account + bucket (~20 min)
5. AWS Organizations → **Add an AWS account** → name it something like `nanza-backup`.
   Give it its own root email and MFA, and don't use it day to day.
6. Sign into the backup account → S3 → **Create bucket**:
   - Pick **a different region** from the media bucket (cross-region, recommended).
   - **Block all public access**: on.
   - **Bucket Versioning**: enable.
   - **Advanced settings → Object Lock**: enable.
   - Create, then **Properties → Object Lock → Default retention**: **Governance**,
     **90 days**.
7. Same bucket → **Management → lifecycle rule**: transition current versions to
   **Glacier Instant Retrieval** after **30** days, and permanently delete noncurrent
   versions after **365** days.

### Part C: replication (~15 min)
8. Back in the main account → media bucket → **Management → Replication rules → Create**:
   - Name: `ugc-backup`, status **Enabled**.
   - **Scope:** limit with a filter, one rule per folder in the "yes" rows above (`post/`,
     `profile/`, `listing/`, `bulk/`, `list/`, `group-banner/`, `group-thumbnail/`,
     `id-verifications/`, `brand/`, `set/`, `supported-tag-value/`,
     `social-link-type/`). **Not `entity/`.**
   - **Destination:** "Specify a bucket in another account" → backup account id + bucket
     name → tick **Change object ownership to destination bucket owner**.
   - **IAM role:** Create new role.
   - **Delete marker replication: leave OFF.** This is what makes it a backup: a delete in
     the main bucket never reaches the copy.
   - Save. When it asks "replicate existing objects?", answer **Yes** → this creates the
     one-time **Batch Replication** job for everything uploaded before today.
9. The wizard shows a **bucket policy** for the backup bucket that lets the replication
   role write to it. Paste it in the backup account → backup bucket → **Permissions →
   Bucket policy**.

### Part D: check it (~5 min)
10. Post a photo in the app. Within a few minutes it shows up in the backup bucket under
    `post/`.
11. **Batch Operations** (main account): the replication job reads **Completed** with 0
    failures.
12. Delete a test object in the main bucket. It's **still in the backup bucket**.

## Restoring
- **One image or a few (Part A):** media bucket → **Show versions** → delete the delete
  marker, or copy the old version over the current one.
- **Many images (Part B/C):** S3 Batch Operations **Copy** from the backup bucket to the
  media bucket, same keys, with a manifest built from the folder plus a time window.
- Then invalidate the image CDN for those paths. Resized variants are cached as immutable.

## Alternative considered
**AWS Backup for S3** (continuous backups + point-in-time restore). It needs versioning
anyway and is nicer for "restore the bucket as of Tuesday". But it costs more per GB and
can't skip `entity/` by prefix. Revisit if we ever want point-in-time restore of the whole
bucket.

## Open questions
1. Same-Region or Cross-Region for the replica? Recommended: Cross-Region. The cost is
   similar at our size, and it also covers a region outage.
2. Noncurrent retention on the media bucket: 60 days or 30?
