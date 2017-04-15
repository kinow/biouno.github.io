---
layout: post
title: "How to use ImageJ with Jenkins"
description: "A tutorial post, about how to use ImageJ with Jenkins"
category:
tags: [jenkins, imagej, tutorial]
author: Bruno P. Kinoshita
---
{% include JB/setup %}


In this post we will go through an example of how to use [ImageJ](https://imagej.nih.gov/ij/)
with Jenkins. You will see that it can be used both as a Build Step, and also as a parameter, with the
[Active Choices Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Active+Choices+Plugin).

## What you will need

You can use any image you would like, but for this example I will use the Christchurch Cathedral picture
from Wikipedia.

<center><img width="500px" src='{{ site.baseurl }}assets/posts/2017-03-05-using-imagej-with-jenkins/chchcathedral.jpg' alt="Christchurch Church" /></center>


The code used for this example can be found in this
[StackOverflow answer](http://stackoverflow.com/a/33177047/1762101). The original code has one image
as input, then outputs two files. The first an inverted image, and the second a grayscale image.

```
package org.biouno.examples;

import java.awt.Color;
import java.awt.image.BufferedImage;

import ij.ImagePlus;
import ij.io.FileSaver;
import ij.process.ImageProcessor;

public class ImageJExample {

    public static void main(String[] args) {
        ImagePlus imgPlus = new ImagePlus("file:///tmp/in.jpg");
        ImageProcessor imgProcessor = imgPlus.getProcessor();

        imgProcessor.invert();
        FileSaver fs = new FileSaver(imgPlus);
        fs.saveAsJpeg("/tmp/out-inverted.jpg");
        BufferedImage bufferedImage = imgProcessor.getBufferedImage();

        for (int y = 0; y < bufferedImage.getHeight(); y++) {
            for (int x = 0; x < bufferedImage.getWidth(); x++) {
                Color color = new Color(bufferedImage.getRGB(x, y));
                int grayLevel = (color.getRed() + color.getGreen() + color.getBlue()) / 3;
                int r = grayLevel;
                int g = grayLevel;
                int b = grayLevel;
                int rgb = (r << 16) | (g << 8) | b;
                bufferedImage.setRGB(x, y, rgb);
            }
        }

        ImagePlus grayImg = new ImagePlus("gray", bufferedImage);
        fs = new FileSaver(grayImg);
        fs.saveAsJpeg("/tmp/out-gray.jpg");
    }

}

```

## Adding a Scriptler Groovy script

Before doing that, please be aware that we will be loading a JAR file dynamically. It can be
slow, result in memory leaks, or even worse, security problems. So please consider using an alternative
approach for production.

You can drop [ImageJ JAR](http://repo1.maven.org/maven2/net/imagej/ij/1.51j/)
in your Jenkins class path, so it will be loaded and available to all your scripts,
or if you are using the scriptler script with a plug-in that supports extending the
class path (such as [Groovy Postbuild Plug-in](https://wiki.jenkins-ci.org/display/JENKINS/Groovy+Postbuild+Plugin))
you can avoid the quick and dirty approach from this post :-)

Here's the Java code re-written in Groovy, loading ImageJ classes dynamically, and
allowing the user to pass input and output locations as parameters.


```
// Need to load the ImageJ classes
import java.io.File
import java.net.URL
loader = this.class.classLoader
loader.addURL( new URL('file:///home/kinow/.m2/repository/net/imagej/ij/1.51j/') )

import java.awt.Color
import java.awt.image.BufferedImage


// If you prefer to use imports, and do not mess with the ClassLoader (which is a great idea too)
// you can drop the jar in Jenkins classpath, or load it via other plug-ins...
// import ij.ImagePlus
// import ij.io.FileSaver;
// import ij.process.ImageProcessor;


//inputFile = "/tmp/gallery/in.jpg"
//outputFile = "/tmp/gallery/out.jpg"

// imgPlus = new ImagePlus("file://${inputFile}");
imgPlusClass = Class.forName("ij.ImagePlus", true, loader)
imgPlus = imgPlusClass.newInstance(["file://${inputFile}"]  as Object[])

imgProcessor = imgPlus.getProcessor();

// fs = new FileSaver(imgPlus);
fsClass = Class.forName("ij.io.FileSaver", true, loader)
fs = fsClass.newInstance([imgPlus]  as Object[])

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

// grayImg = new ImagePlus("gray", bufferedImage);
imgPlusClass = Class.forName("ij.ImagePlus", true, loader)
grayImg = imgPlusClass.newInstance(["gray", bufferedImage]  as Object[])

// fs = new FileSaver(grayImg);
fsClass = Class.forName("ij.io.FileSaver", true, loader)
fs = fsClass.newInstance([grayImg]  as Object[])

fs.saveAsJpeg("${outputFile}");
```
