# FlutterCropImageVideoUI
A custom Flutter widget for cropping images and videos. Users can select aspect ratios (1:1, 16:9, 9:16), scale and move the media, and get crop coordinates. These coordinates can be used with FFmpeg to apply the crop. Ideal for creating media editing apps with customizable crop functionality.

## Dependencies

This Flutter project uses the following dependencies:

- **`cupertino_icons`**: Provides iOS-style icons for a consistent look on Apple devices.
- **`image`**: A package for image manipulation and editing.
- **`image_picker`**: Allows users to pick images from the gallery or camera.
- **`vector_math`**: A library for performing geometric calculations, useful for graphics and animations.
- **`gesture_x_detector`**: Enables the detection of custom gestures in the app.
- **`collision`**: Provides functionality for collision detection in the app.
- **`gallery_saver`**: Saves images or videos to the device's gallery.
- **`ffmpeg_kit_flutter`**: A package for processing video and audio using FFmpeg, supporting features like trimming, conversion, etc.
- **`path_provider`**: Provides access to commonly used file storage locations on the device.
- **`permission_handler`**: Simplifies requesting and managing runtime permissions.

Make sure to add these dependencies to your `pubspec.yaml` file for proper functionality.

## Image Editor App in Flutter

This is a simple Flutter app that allows users to browse an image from their gallery, scale it, and move it within a specific aspect ratio. It also overlays a rectangle that shows the cropping area, which can be dynamically adjusted.

### Code Example

```dart
import 'package:flutter/material.dart';
import 'dart:io';
import 'package:image_picker/image_picker.dart';

class ImageEditorApp extends StatefulWidget {
  @override
  _ImageEditorAppState createState() => _ImageEditorAppState();
}

class _ImageEditorAppState extends State<ImageEditorApp> {
  File? _image;
  double scale = 1.0;
  Offset position = Offset.zero;
  final picker = ImagePicker();

  double rectangleWidth = 200; // Default width
  double rectangleHeight = 200; // Default height
  double aspectRatio = 1.0; // Default aspect ratio

  Future<void> _pickImage() async {
    final pickedFile = await picker.pickImage(source: ImageSource.gallery);
    if (pickedFile != null) {
      setState(() {
        _image = File(pickedFile.path);

        // Dynamically calculate rectangle dimensions
        double imageWidth = 300; // Example width for demonstration
        double imageHeight = 400; // Example height for demonstration
        if (imageWidth > imageHeight) {
          // Landscape: Height matches width
          rectangleHeight = rectangleWidth;
          aspectRatio = rectangleWidth / rectangleHeight;
        } else {
          // Portrait: Width matches height
          rectangleWidth = rectangleHeight;
          aspectRatio = rectangleWidth / rectangleHeight;
        }
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Image Editor")),
      body: Center(
        child: Column(
          children: [
            ElevatedButton(onPressed: _pickImage, child: Text("Browse Image")),
            Expanded(
              child: Stack(
                alignment: Alignment.center,
                children: [
                  if (_image != null)
                    GestureDetector(
                      onScaleUpdate: (details) {
                        setState(() {
                          scale = details.scale;
                          position += details.focalPointDelta / scale;
                        });
                      },
                      child: Transform(
                        transform: Matrix4.identity()
                          ..scale(scale)
                          ..translate(position.dx, position.dy),
                        child: Image.file(_image!),
                      ),
                    ),
                  Positioned(
                    child: Container(
                      width: rectangleWidth,
                      height: rectangleHeight,
                      decoration: BoxDecoration(
                        border: Border.all(color: Colors.red, width: 2),
                      ),
                      child: Center(
                        child: Text(
                          "Ratio: ${aspectRatio.toStringAsFixed(2)}",
                          style: TextStyle(color: Colors.red, fontSize: 16),
                        ),
                      ),
                    ),
                  ),
                ],
              ),
            ),
            if (_image != null)
              Padding(
                padding: const EdgeInsets.all(8.0),
                child: Text(
                  "Outside Pixels: \nX: ${position.dx.abs().toStringAsFixed(2)}, "
                      "Y: ${position.dy.abs().toStringAsFixed(2)}",
                  textAlign: TextAlign.center,
                ),
              ),
          ],
        ),
      ),
    );
  }
}



  
