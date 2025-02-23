# PosterFlags Plugin

A plugin for Jellyfin that enhances poster displays with flag overlays.

## Installation

### Prerequisites
- Ensure you have **.NET 8.0** installed.

### Build Instructions
From the root folder of this project, run the following commands:

```sh
dotnet clean
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






These are my plugin files:

PosterFlagsPlugin.cs:
```
#nullable enable

using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Linq;
using System.Reflection;
using System.Text.RegularExpressions;
using Jellyfin.Plugin.PosterFlags.Configuration;
using MediaBrowser.Common.Configuration;
using MediaBrowser.Common.Plugins;
using MediaBrowser.Controller.Entities;
using MediaBrowser.Controller.Entities.Movies;
using MediaBrowser.Controller.Entities.TV;
using MediaBrowser.Controller.Library;
using MediaBrowser.Model.Plugins;
using MediaBrowser.Model.Serialization;
using Microsoft.Extensions.Logging;
using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Processing;
using System.Drawing;  // Added for Point class

namespace Jellyfin.Plugin.PosterFlags
{
    /// <summary>
    /// PosterFlagsPlugin processes movie and TV show posters, overlaying language flags based on audio languages available.
    /// </summary>
    public class PosterFlagsPlugin : BasePlugin<PluginConfiguration>, IDisposable
    {
        /// <summary>
        /// Gets or sets the singleton instance of the PosterFlagsPlugin.
        /// </summary>
        public static PosterFlagsPlugin? Instance { get; private set; }

        private readonly ILibraryManager _libraryManager;
        private readonly ILogger<PosterFlagsPlugin> _logger;
        private const string BackupFolder = "original_posters";
        private bool _disposed;

        /// <summary>
        /// Initializes a new instance of the PosterFlagsPlugin class.
        /// </summary>
        /// <param name="applicationPaths">The application paths.</param>
        /// <param name="xmlSerializer">The XML serializer used for reading/writing plugin configuration.</param>
        /// <param name="libraryManager">The library manager for interacting with the media library.</param>
        /// <param name="logger">The logger instance for logging messages.</param>
        public PosterFlagsPlugin(IApplicationPaths applicationPaths, IXmlSerializer xmlSerializer, ILibraryManager libraryManager, ILogger<PosterFlagsPlugin> logger)
            : base(applicationPaths, xmlSerializer)
        {
            Instance = this;
            _libraryManager = libraryManager;
            _logger = logger;

            _logger.LogInformation("PosterFlags Plugin loaded. Processing posters...");
            _libraryManager.ItemAdded += OnItemUpdated;
            _libraryManager.ItemUpdated += OnItemUpdated;
        }

        /// <summary>
        /// Gets the name of the plugin.
        /// </summary>
        public override string Name => "PosterFlags";

        /// <summary>
        /// Gets the unique identifier of the plugin.
        /// </summary>
        public override Guid Id => Guid.Parse("4e0813b6-0ff3-494c-bafa-1d5a602834ba");

        /// <summary>
        /// Disposes of the plugin, restoring the original posters and cleaning up resources.
        /// </summary>
        public void Dispose()
        {
            Dispose(true);
            GC.SuppressFinalize(this);
        }

        /// <summary>
        /// Disposes of the plugin, restoring the original posters and cleaning up resources.
        /// </summary>
        /// <param name="disposing">Indicates whether disposal is being performed explicitly.</param>
        protected virtual void Dispose(bool disposing)
        {
            if (_disposed)
            {
                return;
            }

            if (disposing)
            {
                _logger.LogInformation("PosterFlags Plugin unloaded. Restoring original posters...");
                RestoreOriginalPosters();
                _libraryManager.ItemAdded -= OnItemUpdated;
                _libraryManager.ItemUpdated -= OnItemUpdated;
            }

            _disposed = true;
        }

        /// <summary>
        /// Handles item updates and processes posters for movies and TV shows.
        /// </summary>
        /// <param name="sender">The sender object.</param>
        /// <param name="e">The event arguments containing the item change details.</param>
        private void OnItemUpdated(object? sender, ItemChangeEventArgs e)
        {
            if (e.Item is Movie || e.Item is Series)
            {
                ProcessPoster(e.Item);
            }
        }

        /// <summary>
        /// Processes the poster for a given item, overlaying language flags if applicable.
        /// </summary>
        /// <param name="item">The media item (movie or series) to process the poster for.</param>
        private void ProcessPoster(BaseItem item)
        {
            try
            {
                string? posterPath = item.PrimaryImagePath;
                if (string.IsNullOrEmpty(posterPath))
                {
                    return;
                }

                string backupPath = Path.Combine(BackupFolder, Path.GetFileName(posterPath));
                if (!File.Exists(backupPath))
                {
                    Directory.CreateDirectory(BackupFolder);
                    File.Copy(posterPath, backupPath);
                }

                List<string> languages = GetAudioLanguages(item.Path);
                OverlayFlagsOnPoster(posterPath, languages);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error processing poster: {ex.Message}");
            }
        }

        /// <summary>
        /// Extracts the audio languages from the media file using FFmpeg.
        /// </summary>
        /// <param name="mediaPath">The path to the media file.</param>
        /// <returns>A list of audio languages extracted from the media file.</returns>
        private List<string> GetAudioLanguages(string mediaPath)
        {
            List<string> languages = new();
            try
            {
                string output = RunFFmpeg(mediaPath);
                languages = ParseLanguagesFromFFmpeg(output);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error extracting languages: {ex.Message}");
            }
            return languages;
        }

        /// <summary>
        /// Runs FFmpeg to get the media file details and extract audio languages.
        /// </summary>
        /// <param name="mediaPath">The path to the media file.</param>
        /// <returns>The output from FFmpeg containing media details.</returns>
        private string RunFFmpeg(string mediaPath)
        {
            var process = System.Diagnostics.Process.Start(new System.Diagnostics.ProcessStartInfo
            {
                FileName = "ffmpeg",
                Arguments = $"-i \"{mediaPath}\" 2>&1",
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false
            });

            if (process == null)
            {
                throw new InvalidOperationException("Failed to start ffmpeg process.");
            }

            string output = process.StandardOutput.ReadToEnd() + process.StandardError.ReadToEnd();
            process.WaitForExit();
            return output;
        }

        /// <summary>
        /// Parses the audio languages from the FFmpeg output.
        /// </summary>
        /// <param name="output">The FFmpeg output containing media details.</param>
        /// <returns>A list of languages parsed from the FFmpeg output.</returns>
        private List<string> ParseLanguagesFromFFmpeg(string output)
        {
            List<string> languages = new();
            foreach (string line in output.Split('\n'))
            {
                if (line.Contains("Audio") && line.Contains("Stream"))
                {
                    var match = Regex.Match(line, @"\(([a-z]{2,3})\)");
                    if (match.Success)
                    {
                        languages.Add(match.Groups[1].Value);
                    }
                }
            }
            return languages.Distinct().ToList();
        }

        /// <summary>
        /// Overlays the language flags on the poster image.
        /// </summary>
        /// <param name="posterPath">The path to the poster image.</param>
        /// <param name="languages">A list of languages to overlay flags for.</param>
        private void OverlayFlagsOnPoster(string posterPath, List<string> languages)
        {
            using (Image image = Image.Load(posterPath))
            {
                int x = image.Width - (languages.Count * 50);  // Adjust x to fit all flags
                int y = image.Height - 50;  // Place flags at the bottom of the image

                foreach (string lang in languages)
                {
                    using (Image? flag = LoadFlagImage(lang))
                    {
                        if (flag != null)
                        {
                            image.Mutate(ctx => ctx.DrawImage(flag, new Point(x, y), 1f));
                            x += 50;  // Move the position to the right for the next flag
                        }
                    }
                }
                image.Save(posterPath);  // Save the modified poster back to the file system
            }
        }

        /// <summary>
        /// Loads the flag image for a specific language.
        /// </summary>
        /// <param name="lang">The language code for which the flag image should be loaded.</param>
        /// <returns>The loaded flag image, or null if the flag was not found.</returns>
        private Image? LoadFlagImage(string lang)
        {
            string resourceName = $"Jellyfin.Plugin.PosterFlags.Resources.flags.{lang}.png";  // Update this path based on your embedded resource name
            try
            {
                using (Stream? stream = Assembly.GetExecutingAssembly().GetManifestResourceStream(resourceName))
                {
                    if (stream == null)
                    {
                        _logger.LogWarning($"Flag for language '{lang}' not found.");
                        return null;
                    }
                    return Image.Load(stream);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error loading flag for language '{lang}': {ex.Message}");
                return null;
            }
        }

        /// <summary>
        /// Restores the original posters from the backup folder.
        /// </summary>
        private void RestoreOriginalPosters()
        {
            if (!Directory.Exists(BackupFolder))
            {
                return;
            }

            foreach (string backupFile in Directory.GetFiles(BackupFolder))
            {
                string originalPath = Path.Combine("posters", Path.GetFileName(backupFile));
                if (File.Exists(originalPath))
                {
                    File.Copy(backupFile, originalPath, true);
                    File.Delete(backupFile);
                }
            }
        }
    }
}


```

Jellyfin.Plugin.PosterFlags.csproj:
```
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <RootNamespace>Jellyfin.Plugin.PosterFlags</RootNamespace>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Nullable>enable</Nullable>
    <AnalysisMode>AllEnabledByDefault</AnalysisMode>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Jellyfin.Controller" Version="10.9.11" />
    <PackageReference Include="Jellyfin.Model" Version="10.9.11" />
    <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="9.0.2" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="SerilogAnalyzer" Version="0.15.0" PrivateAssets="All" />
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.556" PrivateAssets="All" />
    <PackageReference Include="SmartAnalyzers.MultithreadingAnalyzer" Version="1.1.31" PrivateAssets="All" />
  </ItemGroup>

  <ItemGroup>
    <None Remove="Configuration\configPage.html" />
    <EmbeddedResource Include="Configuration\configPage.html" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="SixLabors.ImageSharp" Version="3.1.6" />
  </ItemGroup>

  <ItemGroup>
      <None Include="Resources\flags\*.png" Pack="true" />
      <EmbeddedResource Include="Resources\flags\*.png" />
  </ItemGroup>


</Project>


```

Configuration/configPage.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>PosterFlags Configuration</title>
</head>
<body>
    <div id="PosterFlagsConfigPage" data-role="page" class="page type-interior pluginConfigurationPage" data-require="emby-input,emby-button,emby-checkbox">
        <div data-role="content">
            <div class="content-primary">
                <form id="PosterFlagsConfigForm">
                    <div class="checkboxContainer checkboxContainer-withDescription">
                        <label class="emby-checkbox-label">
                            <input id="EnableFlagOverlay" name="EnableFlagOverlay" type="checkbox" is="emby-checkbox" />
                            <span>Enable Flag Overlay on Posters</span>
                        </label>
                        <div class="fieldDescription">
                            If enabled, the plugin will overlay flags on posters based on the available audio languages in each movie/TV show.
                        </div>
                    </div>

                    <div>
                        <button is="emby-button" type="submit" class="raised button-submit block emby-button">
                            <span>Save</span>
                        </button>
                    </div>
                </form>
            </div>
        </div>

        <script type="text/javascript">
            var PosterFlagsConfig = { pluginUniqueId: '4e0813b6-0ff3-494c-bafa-1d5a602834ba' };

            document.querySelector('#PosterFlagsConfigPage').addEventListener('pageshow', function() {
                Dashboard.showLoadingMsg();
                ApiClient.getPluginConfiguration(PosterFlagsConfig.pluginUniqueId).then(function (config) {
                    document.querySelector('#EnableFlagOverlay').checked = config.EnableFlagOverlay;
                    Dashboard.hideLoadingMsg();
                });
            });

            document.querySelector('#PosterFlagsConfigForm').addEventListener('submit', function(e) {
                Dashboard.showLoadingMsg();
                ApiClient.getPluginConfiguration(PosterFlagsConfig.pluginUniqueId).then(function (config) {
                    config.EnableFlagOverlay = document.querySelector('#EnableFlagOverlay').checked;
                    ApiClient.updatePluginConfiguration(PosterFlagsConfig.pluginUniqueId, config).then(Dashboard.processPluginConfigurationUpdateResult);
                });

                e.preventDefault();
                return false;
            });
        </script>
    </div>
</body>
</html


```

Configuration/PluginConfiguration.cs
```
// <auto-generated/>
//------------------------------------------------------------------------------
// PluginConfiguration.cs
// Copyright (c) Vitrua. All rights reserved.
// Licensed under the MIT License. See LICENSE file in the project root for full license information.
//------------------------------------------------------------------------------

namespace Jellyfin.Plugin.PosterFlags.Configuration
{
    using MediaBrowser.Model.Plugins;

    /// <summary>
    /// Plugin configuration.
    /// </summary>
    public class PluginConfiguration : BasePluginConfiguration
    {
        /// <summary>
        /// Initializes a new instance of the <see cref="PluginConfiguration"/> class.
        /// </summary>
        public PluginConfiguration()
        {
            // Default settings
            EnableFlagOverlay = true;
        }

        /// <summary>
        /// Gets or sets a value indicating whether the flag overlay feature is enabled.
        /// </summary>
        public bool EnableFlagOverlay { get; set; }
    }
}


```


In Resources/flags/ there are at the moment eng.png, fra.png, ger.png, jpn.png, spa.png, ita.png, zho.png, kor.png

The Jellyfin plugin that when enabled, it copies the original poster of movies and tvshows in a temporary 'original' folder, then it substitutes the main displayed poster with one with a small flag in the corner for every language it finds in the audio of the files. This way the new poster will be visible from all the clients. It doesn't process the poster if it has been converted already. 
SO there will be a single poster with all the flags of the available languages one next to the other. When the plugin is disabled it must revert the process, so it must delete the poster with flag/flags and reextract the backupped poster.
I want a self-contained plugin, embed the images of the flags.

BUT:

dotnet build
MSBuild version 17.8.5+b5265ef37 for .NET
  Determining projects to restore...
  All projects are up-to-date for restore.
/home/giacomo/projects/posterflags-plugin/PosterFlagsPlugin.cs(25,18): error CS1591: Missing XML comment for publicly visible type or member 'PosterFlagsPlugin' [/home/giacomo/projects/posterflags-plugin/Jellyfin.Plugin.PosterFlags.csproj]
/home/giacomo/projects/posterflags-plugin/PosterFlagsPlugin.cs(27,42): error CS1591: Missing XML comment for publicly visible type or member 'PosterFlagsPlugin.Instance' [/home/giacomo/projects/posterflags-plugin/Jellyfin.Plugin.PosterFlags.csproj]
/home/giacomo/projects/posterflags-plugin/PosterFlagsPlugin.cs(33,16): error CS1591: Missing XML comment for publicly visible type or member 'PosterFlagsPlugin.PosterFlagsPlugin(IApplicationPaths, IXmlSerializer, ILibraryManager, ILogger<PosterFlagsPlugin>)' [/home/giacomo/projects/posterflags-plugin/Jellyfin.Plugin.PosterFlags.csproj]
/home/giacomo/projects/posterflags-plugin/PosterFlagsPlugin.cs(45,32): error CS1591: Missing XML comment for publicly visible type or member 'PosterFlagsPlugin.Name' [/home/giacomo/projects/posterflags-plugin/Jellyfin.Plugin.PosterFlags.csproj]
/home/giacomo/projects/posterflags-plugin/PosterFlagsPlugin.cs(46,30): error CS1591: Missing XML comment for publicly visible type or member 'PosterFlagsPlugin.Id' [/home/giacomo/projects/posterflags-plugin/Jellyfin.Plugin.PosterFlags.csproj]


BUILD FAILED