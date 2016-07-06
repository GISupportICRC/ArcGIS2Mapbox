# ArcGIS2Mapbox
Script to convert from shapefile to Mapbox-hosted vector tile layers using tippecanoe, by [Development Seed](https://developmentseed.org/)

## Overview
This script converts a shapefile or zipped shapefile archive to Mapbox-hosted vector tile layers, which are named after the input files and which overwrite/update the remote layers if they already exist. Vector tile layers are optimized for efficient streaming and display at multiple zoom levels.

## Procedure
This script begins by converting a shapefile to WGS-84-projected GeoJSON, which is converted to optimized vector tile layers using Mapbox's [tippecanoe](https://github.com/mapbox/tippecanoe) processing tool. The script then follows the procedure described in the Mapbox [upload API documentation](https://www.mapbox.com/api-documentation/#uploads) for Python. As a secondary parameter, a max zoom level may be specified, which defines the Mapbox zoom level at which the output will be rendered at full detail. A zoom level of 10 is recommended for polygons and lines in order to enable rendering of the data online, although this parameter can be adjusted upward at the expense of processing time. For points, it is recommended that tiling be skipped by omitting the max zoom level parameter.

#### The script's workflow is roughly as follows:
- Unzip shapefile archive, if necessary.
- Convert shapefile to WGS-84 projected GeoJSON using OGR and OSR.
- Generate vector tile layers (.mbtile format) using tippecanoe.
- Poll Mapbox's API for the credentials of an Amazon S3 bucket in which to stage the upload.
- Upload to the staging bucket.
- Send staged data to Mapbox's Uploader API.

If no max zoom level is specified, the script will skip the GeoJSON and tiling step, zip the shapefile support files, and send the archive directly to the Mapbox Uploader API.

## Installation prerequisites
In order to generate vector tiles, which are a special spatial format that streams and displays efficiently at multiple zoom levels by only loading in vertices as necessary, this script depends on a separate installation of Mapbox's [tippecanoe](https://github.com/mapbox/tippecanoe) processing tool. The installation is extremely easy on Mac OS, but on Windows it requires a roughly 10 minute installation process, and the script must be executed through a program called [Cygwin](https://www.cygwin.com/).




#### Installing tippecanoe on Windows
On Windows, a version of tippecanoe must be specially compiled from its source code so that it can run inside a program called Cygwin, which provides Linux-like functionality through a collection of Windows-native utilities (i.e., non-emulated).

1. Download [setup-x86_64.exe](http://cygwin.com/setup-x86_64.exe) from the Cygwin website.
2. There are several development tools that will need to be installed as Cygwin extensions. Fortunately, the Cygwin setup application can be operated from the Windows command line and it can download support files as requested, so it won't be necessary to install the components individually. Navigate to the Windows Download directory by typing

    `cd %userprofile%/Downloads`,

    then enter the following command to install Cygwin along with the necessary development tools:

    `setup-x86_64.exe -q -P zlib-devel,libsqlite3-devel,gcc-g++,make,python,git,gdal,python-gdal`

3. Open the newly installed Cygwin64 Terminal application when installation is complete.
4. Ensure the Proxy are setup for the installation 
-- proxy= XXXXX.icrc.priv:8080
-- export https_proxy=http://XXXXX.icrc.priv:8080
-- export http_proxy=http://XXXXX.icrc.priv:8080



5. Install the Python Package Manager by typing

    `python -m ensurepip`

    in the Cygwin terminal.
6. Clone the tippecanoe sourcecode from Github, by entering:

    `git clone https://github.com/mapbox/tippecanoe.git`

7. In Windows, the tippecanoe source code will have been saved within your Cygwin home directory at `C:\cygwin64\home\{your username}\tippecanoe`. Navigate to this directory and open the Makefile within using Wordpad (Notepad will format the line endings incorrectly).
8. Add the text `-U__STRICT_ANSI__` to the line reading `CXXFLAGS := $(CXXFLAGS) -std=c++11`, so that it reads:

    `CXXFLAGS := $(CXXFLAGS) -std=c++11 -U__STRICT_ANSI__`

9. Back in the Cygwin terminal, enter the tippecanoe directory by typing

    `cd tippecanoe`.

10. Compile the tippecanoe source code by typing

    `make`.

11. Finally, install the compiled program by typing

    `make install`.

Tippecanoe is now ready for use, and the source directory at `C:\cygwin64\home\{your username}\tippecanoe` can be deleted if desired.

## Script installation
#### Clone script from Github
Check out a local copy of the script using the command
```
git clone https://github.com/developmentseed/icrc-mb-sync.git
```
then `cd icrc-mb-sync` to enter its directory.

#### Install Python dependencies (both platforms)
This script requires the additional Python dependencies of `boto3`, `cachecontrol`, `requests`, and `uritemplate.py`, as listed in `requirements.txt`. To install, run the command
```
pip install -r requirements.txt
```
from within the script directory.

#### Obtain a Mapbox private key
Programmatic access to the Mapbox Upload API requires a Mapbox private key with write permissions, which is not the default. This key can be generated by choosing to add an access token in the online [Mapbox account manager](https://www.mapbox.com/studio/account/tokens/), and enabling the optional secret token scopes through the Token Scopes accordion menu.

## Script usage
#### Execute from command line (Cygwin CLI on Windows):
The uploader script takes as arguments an input shapefile or zipped shapefile, a Mapbox private key, and a max zoom level. For lines and polygons, a max zoom level of at least 10 is recommended. For points, it is recommended that tile generation be skipped by omitting the max zoom level parameter. The script will convert the input data to vector tile layers, and will update any Mapbox-hosted tile layers having the same name within the account associated with the input key. Enter the following command to execute, either while inside the script's directory or by providing the absolute rather than relative paths to the script and input directory:
```
python mapbox_uploader.py {input shapefile} {Mapbox token} {max zoom level, optional}
```
**i.e.:** `python mapbox_uploader.py input.shp sk.eyJ1IjoibmJ1bWJhcmciLCJ... 10`
**i.e.:** `python mapbox_uploader.py ../MapboxLayers/input.shp sk.eyJ1IjoiaWNyYyIsImEiOiJjaXBqdDd 10`

Cygwin provides connections to drives outside of its internal filesystem through a mount point called /cygdrive/
```
python Script_Shp2Mapbox.py "/cygdrive/d/Projets/Geoportal/Server Objects/Data/Temp/Mapbox/MapboxLayers/input.shp"   sk.eyJ1IjoiaWNyYyIsImxxxxx 10
```
Notice the quotes surrounding the path- quotes are necessary when referencing paths that contain spaces, or else each unbroken part of the string will be interpreted as a separate, nonsensical command. Also, in a Python script, if a string contains quotes it must be wrapped in a different kind of quotes, such as single quotes.



#### Execute from Python scripts (Cygwin CLI on Windows):
This script can be executed from other scripts in the same directory by simply importing it, as long as the calling environment is the same as the script's processing environment (i.e., as long as the calling script is also running in either Cygwin or Mac OS). For calling instructions from a program that is running on Windows instead, see below.
```python
import arc2mb
arc2mb({input directory}, {Mapbox token}, {max zoom level})
```

#### Execute from Windows command line:
Scripts in the Cygwin environment can be executed from the Windows command line or through Windows batch commands by providing the path to Cygwin's bash (Linux command line) executable as the first argument, followed by the `--login` flag, followed by the `-c` flag, followed by a text string of the same command you would use within Cygwin using absolute paths, as shown below:
```
c:\cygwin64\bin\bash --login -c "python {abs. path to script} {abs. path to input shp.} {Mapbox token} {max zoom level}"
```
**i.e.:** `c:\cygwin64\bin\bash --login -c "python ~/ArcGIS2Mapbox/arc2mb.py ~/icrc_data/input.shp sk.eyJ1IjoibmJ1bWJhcmciLCJ... 10"`






#### Execute from Windows Python scripts (i.e. scripts tied to arcpy):
Similarly, it is possible to directly call the script through the Cygwin shell from within Python scripts that are running on the Windows environment, such as through the example below:
```python
import subprocess
subprocess.call([
    "c:\cygwin64\bin\bash", "--login", "-c",
    "python arc2mb.py {abs. path to script} {abs. path to input shp.} {Mapbox token} {max zoom level}"
])
```
