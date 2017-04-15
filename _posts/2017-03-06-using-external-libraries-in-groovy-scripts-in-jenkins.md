---
layout: post
title: "Using external libraries in Groovy scripts in Jenkins"
description: "A short post about using external libraries in Groovy scripts in Jenkins"
category:
tags: [jenkins, imagej, tutorial]
author: Bruno P. Kinoshita
---
{% include JB/setup %}

In this post I will show two techniques I used in the past to use external libraries in
Groovy scripts in Jenkins. The first is a cleaner way, where you won't mess so much
with your Jenkins installation if you know what you are doing. And the second way is
more dangerous and not really recommended, but that may be useful to know that this option
exists too.

## Adding an extra JAR in Jenkins libraries folder

As the section title goes, this method is very straightforward. If you are running Jenkins
with the ol' `java -jar jenkins.war`, then you will have a *$JENKINS_HOME* directory,
which defaults to *$USER/.jenkins/*.

Inside your Jenkins home directory, you should find a directory **war**, with another subdirectory
called **lib**. You can drop your library JARs there, and restart Jenkins. After that, the classes
in that JAR will be loaded automatically for you.

```
import java.awt.Color
import java.awt.image.BufferedImage
import ij.ImagePlus
import ij.io.FileSaver;
import ij.process.ImageProcessor;


inputFile = "/home/kinow/Downloads/gallery/in.jpg"
outputFile = "/home/kinow/Downloads/gallery/out.jpg"

imgPlus = new ImagePlus("file://${inputFile}");
imgProcessor = imgPlus.getProcessor();

fs = new FileSaver(imgPlus);
bufferedImage = imgProcessor.getBufferedImage();

for (int y = 0; y < bufferedImage.getHeight(); y++) {
  for (int x = 0; x < bufferedImage.getWidth(); x++) {
    color = new Color(bufferedImage.getRGB(x, y));
    grayLevel = (int) (color.getRed() + color.getGreen() + color.getBlue()) / 3;
    r = (int) grayLevel;
    g = (int) grayLevel;
    b = (int) grayLevel;
    rgb = (r << 16) | (g << 8) | b;
    bufferedImage.setRGB(x, y, rgb);
  }
}

grayImg = new ImagePlus("gray", bufferedImage);

fs = new FileSaver(grayImg);
fs.saveAsJpeg("${outputFile}");
```

