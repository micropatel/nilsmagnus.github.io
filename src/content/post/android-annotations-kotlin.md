+++
date = "2017-09-26T19:56:52+01:00"
draft = false
title = "Converting an android project to kotlin"
featuredImage = "/kotlin-plugin-bugging.png"
tags        = [ "android", "kotlin", "android-annotations"]
+++

This is a small write-up of my experiences from converting a native android-app from java to kotlin. 

## TLDR;

1. Convert the project using android studio
2. Handle nullability-compilation errors
3. Android annotations 
    1. build script
    2. apply lateinit on @ViewById
4. Convert your data-objects into small classes, go for immutable classes
5. Kotlin is cool, and immutability rocks. Bugs in android-studio shows that it is still early days for kotlin-development on android.

# Convert your project!

The first step is easy, android-studio will do most of the work for you. 
To convert the existing javacode into kotlin, simply __select the src/main/java__ folder in the project and choose
__Code->"Convert Java File to Kotlin File"__. Android studio will then try as best as it can
to convert all your java-code to kotlin-code. 

![](/kotlin-convert.png "Convert to kotlin in android-studio")

Android studio will convert all your .java files into .kt files in-place, leaving them in src/main/java. 

Your project will probably not compile now, because of differences in how java and kotlin handles nullability. 

# nullability(.!!?) issues

One of the big advantages kotlin has over java is the handling of nullability. By default, an object is non-nullable and has to explicitly specified to be nullable.


In my case, android-studio generated code that would not compile. My data-objects were converted to have nullable-fields, but several kotlin-functions specified non-nullable arguments. To fix this, simply allow nullable-arguments or, even better, make your data-objects immutable with non-nullable fields!

_error after automatic conversion in android-studio_ :

![nullability error after conversion](/nullability-error1.png "error after automatic conversion in android-studio")

The generated nullable field in the class "Shipment:"

    class Shipment {
        var shipmentNumber: String? = null
        ...
        
Simply change this class to an immutable version, by specifying the arguments in a constructor:

    class Shipment(val shipmentNumber: kotlin.String, /* other fields goes here */)






# Android-annotations (if applicable)

When android-studio converted your project, it most likely broke
the usage of your android-annotations usage. 

To use android-annotations, add the _kotlin-kapt_ plugin to your build-script and add the android-annotations-dependency to the kapt-configuration:

    ...
    apply plugin: 'com.android.application'
    apply plugin: 'kotlin-android'
    apply plugin: 'kotlin-kapt'
    ...

    dependencies{
    ...
        kapt "org.androidannotations:androidannotations:4.3.1"

    }


The fields marked with _@ViewById_ will make the annotation-processor complain with the following error:

    org.androidannotations.annotations.ViewById cannot be used on a private element

To resolve this, change the generated code:
    
    @ViewById(R.id.adView)
    var adView: AdView? = null


by adding lateinit and removing the nullability:

    @ViewById(R.id.adView)
    lateinit var adView: AdView


The __lateinit__ keyword in kotlin means that the field can be treated as 
non-nullable, but will be initialized after the constructor has been called 
(for the purpose of dependency-injection and lazy-initialization).

After removing the nullability-option on the view-fields, the "_!!_" notation on references to these fields can be removed. 


# Other thoughts

Even if it is several months since the announcement for kotlin-support on android, the kotlin-plugin
for android studio seems somewhat immature. During my little experience with kotlin in android-studio the plugin 
 has crashed several times and I had to restart android studio to continue working. 
 
 
 ![plugin-crash and restart warning](/kotlin-plugin-bugging.png)

 
 I also tried to change the folder-name for my kotlin source-code from _src/main/java_ to _src/main/kotlin_, but this
 caused android studio to stop giving lint-hints and auto-completion even if some of the code was highlighted properly.
  
These two issues makes me think that kotlin might still be in its early days on android and that more improvements can be done to 
improve developer satisfaction. 


However, the advantages that kotlin brings with easier-to-read code and immutability
are welcome features that will help avoiding bugs and ease both maintenance and writing of apps for android. A simple(and a bit naive) measure of maintanability is lines of code(LOC), in my case I reduced the LOC by about 33%:

| LOC Before(java)| LOC After(kotlin)|
|:-------------:|:-------------:|
| 6540 | 4389|

   
