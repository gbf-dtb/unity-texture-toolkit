Fork of esterTion's [Unity Texture Toolkit](https://github.com/esterTion/unity-texture-toolkit).

The only thing I changed was the main.php for the Re:Dive tools.  
Cygames changed the manifest format in 2024 April (along with the Unity update) and added one new column, from this:

    bundle filename, bundle md5 hash, bundle type, bundle size  
...to this:

    bundle filename, bundle md5 hash, bundle xxHash64 hash, bundle type, bundle size

And since they're shifted by one column, +1 to the manifest array to get `bundleHash` and `bundleSize`.

On top of that, the URL now uses XXH64 hash (64-bit, 16 chars) instead of MD5 hash (128-bit, 32 chars), so instead of using `md5($bundle)` to check for hash, `hash("xxh64", $bundle)` is used instead.

*※take note that support for xxHash was added only in [PHP 8.1](https://www.php.net/manual/en/migration81.new-features.php#xxHash) so if you're using an older version, it'll not work.*

---

# Unity Texture Toolkit

> written in PHP

## What's this for
I wrote this for automatic game resource updating and dumping to png/webp, it's currently deployed on my server https://redive.estertion.win/  
Main codes are highly inspired by [AssetStudio](https://github.com/Perfare/AssetStudio)  

## Requirement
[php-ext-lz4](https://github.com/kjdev/php-ext-lz4/) ( For decompressing the bundle )  
[astcenc](https://github.com/ARM-software/astc-encoder/tree/master/Binary) ( For decompressing ASTC texture format )  
[ffmpeg](http://ffmpeg.org/download.html) ( For reencoding )  
`astcenc` and `ffmepg` need to be placed under PATH, like `/usr/bin` (cron may not search for `/usr/local/bin`, or you can configure it)

## Limitation
Current supported format:
- ASTC compressed 2d image
- Raw rgb ( e.g. RGB24 RGB565 RGBA4444 etc )
> I've only met these formats so far

## How to use

```PHP
require_once 'UnityAsset.php'; // This will require UnityBundle.php as it needs FileStream

$bundleFileName = 'bundle_name.unity3d'; // This is the bundle file with file header "UnityFS"
$bundleFileStream = new FileStream($bundleFileName); // Create a read stream
$assetsList = extractBundle($bundleFileStream); // This will extract assets to disk
unset($bundleFileStream); // Free the handle

foreach ($assetsList as $asset) {
  if (substr($asset, -4,4) == '.resS') continue; // .resS file is external data storage file
  $asset = new AssetFile($asset);

  foreach ($asset->preloadTable as &$item) {
    if ($item->typeString == 'Texture2D') {
      $item = new Texture2D($item, true); // Parse and read data
      $item->exportTo($item->name, 'webp', '-lossless 1'); // export to format, with additional encode parameters
      // $item->exportTo($item->name, 'png');
      unset($item); // Free up memory
    }
  }
  $asset->__desctruct();
  unset($asset); // Free up memory
}
foreach ($assetsList as $asset) {
  unlink($asset); // clean up files
}
```

### What's in the sub directory
These are the files I'm croning on my server for automatic update  
There are different versions, as I wrote the `Princess Connect Re:dive` first, its code has least feature, and `cgss` is the current latest  
They all share the same `UnityBundle.php`

### Implementation detail
1. UnityBundle.php

class `FileStream($filename)` and `MemoryStream($data)`:  
Two similar stream reader, FileStream accepts filename, MemoryStream accepts string as binary.  
Reading value can perform either function `$stream->readInt32()` or property `$stream->ulong`  
Property `position` can get current and set to seek point  
Property `littleEndian` determine either use little endian reading or not  
Function `write($newData)` will always write to the end for MemoryStream, but can overwrite current data position for FileStream  

function `extractBundle($bundleStream)`:  
Accepts a stream, may throw exception if is invalid file, or chunk is LZMA compressed, or something wrong happened  
Returns a list of asset file names extracted

2. UnityAsset.php

class `AssetFile($assetFileName)`:  
Accepts a filename, may throw exception if something is not supported  
Can get resource info through property `preloadTable`  

class `Texture2D($preloadData, $readSwitch = false)`:  
Accepts an `AssetPreloadData` item from `preloadTable`, may throw an exception if format not supported  
Second parameter determine weither read data or not, if you only want to get the information  
Can call member function `exportTo($saveTo, $format = 'png', $extraEncodeParam = '')` to export supported texture
