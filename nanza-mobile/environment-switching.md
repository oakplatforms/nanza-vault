# Environment Switching Guide

This document outlines the process for switching between different environments (dev, stage, prod) in the Nanza Mobile app.

## Overview

The app uses multiple environments with different configurations:
- **Amplify Environments**: dev, stage, prod
- **Environment Variables**: S3_BASE_URL and API_BASE_URL (with dev, stage, prod variants)

## Environment Variables

The app uses two main environment variables that need to be configured for each environment:

- `S3_BASE_URL` - Base URL for S3 storage
- `API_BASE_URL` - Base URL for API endpoints

Each environment has its own values for these variables.

## Switching Environments

### Step 1: Switch Amplify Environment

```bash
amplify env checkout <environment-name>
```

Available environments:
- `dev`
- `stage` 
- `prod`

Example:
```bash
amplify env checkout dev
```

### Step 2: Update Environment Variables

After switching the Amplify environment, you need to manually update the environment variables in your configuration files.

**Note**: You'll need to uncomment the appropriate environment variables for the environment you're switching to and comment out the others.

### Step 3: Reset Metro Bundler

After making environment changes, you must reset the Metro bundler to ensure the new configuration is loaded:

```bash
npx react-native start --reset-cache
```

## Complete Workflow Example

Here's the complete process for switching from `dev` to `stage`:

1. **Switch Amplify environment:**
   ```bash
   amplify env checkout stage
   ```

2. **Update environment variables** (manually edit config files):
   - Comment out dev environment variables
   - Uncomment stage environment variables

3. **Reset Metro bundler:**
   ```bash
   npx react-native start --reset-cache
   ```

## Troubleshooting

### Common Issues

- **Metro cache issues**: Always run `--reset-cache` after environment changes
- **Environment variable not loading**: Ensure you've commented/uncommented the correct variables
- **Amplify sync issues**: Run `amplify status` to check for any pending changes

### Verification

To verify you're on the correct environment:

```bash
# Check current Amplify environment
amplify env list

# Check environment status
amplify status
```

## Best Practices

1. **Always reset Metro cache** after environment changes
2. **Verify environment variables** are correctly set before testing
3. **Use consistent naming** for environment variables across all environments
4. **Document any environment-specific configurations** in this file

## Environment Configuration Files

The environment variables are typically configured in:
- `.env` files (if using react-native-dotenv)
- Configuration files in the app's config directory
- Amplify configuration files

**Note**: The exact location and format of these files should be documented based on your specific setup. 