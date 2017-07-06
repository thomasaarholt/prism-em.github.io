# Prismatic Packaging Instructions

These are the steps necessary to create Prismatic packages from the source code

## Mac OS X

*Currently, Macs do not contain NVIDIA GPUs, so there is only a CPU version of the application bundle*

From an existing version of the prismatic-gui.app bundle, simply copy the latest `prismatic-gui` into Contents/MacOS

Should you need to remake the bundle, it should contain the following directory structure
Contents/
&nbsp;&nbsp;&nbsp;&nbsp;Info.plist
&nbsp;&nbsp;&nbsp;&nbsp;MacOS/
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;prismatic-gui
&nbsp;&nbsp;&nbsp;&nbsp;Resources/
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;prismatic-gui.icns

Where prismatic-gui.icns was made following [this blog post](https://blog.macsales.com/28492-create-your-own-custom-icons-in-10-7-5-or-later) using Inkscape to generate the images and `iconutil` to convert the iconset to a .icns.  

 Info.plist is a basic XML file with the following contents


~~~
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleExecutable</key>
  <string>prismatic-gui</string>
  <key>CFBundleName</key>
  <string>prismatic-gui</string>
  <key>CFBundleIconFile</key>
  <string>prismatic-gui.icns</string>
</dict>
</plist>
~~~

## Windows 10

Configure/compile with `PRISMATIC_ENABLE_GUI=1` with CMake + Visual Studio (I use MSVS 2015) 64-bit. Set to Release mode and build the solution. Then you must copy the following .dll files into the Release directory containing prismatic-gui.exe

* libfftw3f-3.dll
* Qt5Core.dll
* Qt5Widgets.dll
* Qt5Gui.dll

Colin indicated that it may also be necessary to include the plugins/platforms/ folder from within Qt 5. I did not need this folder when compiling and running on machines all using Windows 10, but maybe this folder is necessary when distributing to older versions, so I include it to be safe.

To add the icon to the executable, open the solution in Visual Studio. Open the resources view `view -> Other Windows -> Resources View`, right click `prismatic-gui` and select `Add -> import`, choose the ".ico" file, and recompile. The ".ico" file was generated from a .png image using the online resource [convertIcon!](www.converticon.com)

