# IDE Configuration

## Introduction

If you don't see WordPress native function suggestions in your IDE, it's likely due to incomplete PHP include path configuration. Here's how to configure the main IDEs to resolve this issue.

## PhpStorm

1. Go to `File > Settings` (Windows/Linux) or `PhpStorm > Settings` (macOS)
2. Navigate to `PHP` configuration
3. Click on the `Include Path` tab
4. Add the following paths using the `+` button:
   - `public/cms`
   - `vendor`
5. Click `Apply` then `OK`
6. Restart PhpStorm for changes to take effect

## Visual Studio Code

1. Open Settings with `Ctrl+,` (Windows/Linux) or `Cmd+,` (macOS)
2. Search for "php include path"
3. Under `PHP > Include Path`, add the following paths:
   ```json
   {
     "php.includePath": [
       "./public/cms",
       "./vendor"
     ]
   }
   ```
4. Restart VS Code for changes to take effect

## Important Notes

- Paths should be relative to your project root
- After any include path modifications, it's recommended to restart your IDE
- If you're using specific PHP plugins in VS Code (like PHP Intelephense), you might need to configure include paths in their respective settings as well

## Verification

To verify the configuration is correct:

1. Open a PHP file in your project
2. Try using a WordPress native function (e.g., `get_post()`)
3. Autocomplete should now suggest the function and display its documentation

If suggestions still don't appear, verify that:
- Include paths are correctly written
- Your IDE has been restarted
- Your IDE's PHP plugins are up to date