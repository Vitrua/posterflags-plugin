# PosterFlags Plugin

A plugin for Jellyfin that enhances poster displays with flag overlays.

## Installation

### Prerequisites
- Ensure you have **.NET 8.0** installed.

### Build Instructions
From the root folder of this project, run the following commands:

```sh
dotnet clean
dotnet restore
dotnet build
dotnet build --configuration Release
```

### Deploying the Plugin
Once compiled, the plugin DLL will be generated at:

```
bin/Release/net8.0/Jellyfin.Plugin.PosterFlags.dll
```

To install it:
1. Copy `Jellyfin.Plugin.PosterFlags.dll` to your Jellyfin config folder at:  
   ```
   config/data/plugins/PosterFlags
   ```
2. Restart your Jellyfin server to apply the changes.