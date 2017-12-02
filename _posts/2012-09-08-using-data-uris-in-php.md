---
layout: post
title:  "Using data URIs in PHP"
date:   2012-09-08 00:00:00
categories: php datauri
---

I'm currently working on a JavaScript application that relies upon the HTML Canvas Element's toDataURL function to transfer the canvas' image data to my PHP API. However finding little functionality in PHP for dissecting/constructing data URIs I decided to write my own PHP class for handling them.

You can find a link to the class below, along with some examples of how to use it.

## Download

[View the DataUri PHP class on my GitHub account](https://gist.github.com/3661056).

*The DataUri class should not be used for validation! I developed it for convenience, not security.*

## Usage examples

Below are a number of examples showing how the DataUri class can used for reading and constructing data URIs.

### Data URI to image

The main reason for creating the class. Below shows how the class can be used to parse a data URI containing image data - in my case it was a data URI produced by the HTML Canvas Element's [toDataURL()](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toDataURL) function.

```php
$inputValue =
  'data:image/gif;base64,R0lGODlhFwAQAIAAAAAAAP///yH5BAAAAAAA'.
  'LAAAAAAXABAAAAI3DIKJp73vHEvgMGsgpuviXUXNJImdGJpqWW3Zc1FfBo'.
  'agdsJGu0fk6gMKhzjOrEQz9kbFWFJRAAA7';

// Parse data URI string
$dataUri = null;
if(DataUri::tryParse($inputValue, $dataUri)) {

  // Attempt to decode URI's data
  $data = false;
  if($dataUri->tryDecodeData($data)) {

    // Create and output image
    $image = imagecreatefromstring($data);
    if ($image !== false) {
      header('Content-Type: image/gif');
      imagegif($image);
      imagedestroy($image);
    }
  }
}
```

### Image to data URI

Easily convert an image to a data URI, which in this instance is being used for an Image element.

```php
$fileContents = file_get_contents('./open_rights_group_logo.png');

if($fileContents !== false) {
  $dataUri = new DataUri(
    'image/png',
    $fileContents,
    DataUri::ENCODING_BASE64
  );

  echo '<img src="'.$dataUri->toString().'" alt="Open Rights Group logo" />';
}
```

### Font to data URI

Use the DataUri class to easily generate a data URI to store a font-face in CSS.

```php
$fontFile = file_get_contents('./MyFont.ttf');
$dataUri = new DataUri('font/opentype', $fontFile, DataUri::ENCODING_BASE64);

echo <<<EOT
@font-face {
    font-family: "My Font";
    src: url("{$dataUri}");
}
EOT;
```
