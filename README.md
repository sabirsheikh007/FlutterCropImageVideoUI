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

## Custome Crop UI App in Flutter

This is a simple Flutter app that allows users to browse an image from their gallery, scale it, and move it within a specific aspect ratio. It also overlays a rectangle that shows the cropping area, which can be dynamically adjusted.
```dart
import 'dart:io';
import 'package:ffmpeg_kit_flutter/ffmpeg_kit.dart';
import 'package:ffmpeg_kit_flutter/ffmpeg_kit_config.dart';
import 'package:ffmpeg_kit_flutter/return_code.dart';
import 'package:image/image.dart' as img;
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';

class ProjectTree extends StatefulWidget {
  const ProjectTree({Key? key}) : super(key: key);

  @override
  State<ProjectTree> createState() => _ProjectTreeState();
}

class _ProjectTreeState extends State<ProjectTree> {
  File? file;
  String? filePath;
  double? imageWidth;
  double? imageHeight;

  void BrowsFile() async {
    final ImagePicker _picker = ImagePicker();
    final XFile? pickedFile = await _picker.pickImage(source: ImageSource.gallery);

    if (mounted && pickedFile != null) {
      setState(() {
        file = File(pickedFile.path);
        filePath=pickedFile.path;
        // Load the image to get its width and height
        loadImage(file!);
      });
    }
  }

  Future<void> loadImage(File imageFile) async {
    final img.Image image = img.decodeImage(imageFile.readAsBytesSync())!;
    setState(() {
      imageWidth = image.width.toDouble();
      imageHeight = image.height.toDouble();
    });
    print('Image Width: $imageWidth, Image Height: $imageHeight');
    Navigator.push(
      context,
      MaterialPageRoute(builder: (context) => ImageCropper(file: file, imageWidth: imageWidth, imageHeight: imageHeight, filePath: filePath,)),
    );
  }
  Future<void> requestPermissions() async {
    PermissionStatus status = await Permission.storage.request();
    if (status.isGranted) {
      print("Permission granted");
    } else {
      print("Permission denied");
    }
  }
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('demo'),),
      body:  Column(
        children: [
          Container(
            child: ElevatedButton(onPressed: BrowsFile, child: Text('File')),
          ),
          Container(
            child: ElevatedButton(onPressed: requestPermissions, child: Text('Storage Permission')),
          ),
        ],
      ),
    );
  }
}

class ImageCropper extends StatefulWidget {
  final File? file;
  final String? filePath;
  final double? imageWidth;
  final double? imageHeight;

  const ImageCropper({
    Key? key,
    required this.file,
    required this.imageWidth,
    required this.imageHeight,
    required this.filePath,
  }) : super(key: key);

  @override
  _ImageCropperState createState() => _ImageCropperState();
}

class _ImageCropperState extends State<ImageCropper> {
  Rect? cropRect;
  String selectedAspectRatio = "1:1"; // Default aspect ratio
  late BoxConstraints constraints;
  late TransformationController _transformationController;
  double zoomScale = 1.0;
  Offset panOffset = Offset(0, 0); // Track pan offset
  String _croppedImagePath = '';

  @override
  void initState() {
    super.initState();
    _transformationController = TransformationController();
    _croppedImagePath = '${Directory.systemTemp.path}/cropped_image.jpg'; // Path to save the cropped image
  }

  @override
  void dispose() {
    _transformationController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        title: const Text('Custom Cropper'),
        actions: [
          if (cropRect != null)
            IconButton(
              icon: const Icon(Icons.check),
              onPressed: () {
                //==================impiment hear===========
                double scaleX = constraints.maxWidth / widget.imageWidth!;
                double scaleY = constraints.maxHeight / widget.imageHeight!;
                double scale = scaleX < scaleY ? scaleX : scaleY;

                double scaledImageWidth = widget.imageWidth! * scale;
                double scaledImageHeight = widget.imageHeight! * scale;

                double offsetX = (constraints.maxWidth - scaledImageWidth) / 2;
                double offsetY = (constraints.maxHeight - scaledImageHeight) / 2;

                double originalLeft = (cropRect!.left - offsetX) / scale;
                double originalTop = (cropRect!.top - offsetY) / scale;
                double originalWidth = cropRect!.width / scale;
                double originalHeight = cropRect!.height / scale;

                originalLeft = originalLeft.clamp(0.0, widget.imageWidth!);
                originalTop = originalTop.clamp(0.0, widget.imageHeight!);
                originalWidth = originalWidth.clamp(0.0, widget.imageWidth! - originalLeft);
                originalHeight = originalHeight.clamp(0.0, widget.imageHeight! - originalTop);

                print("Crop Area in Original Image Dimensions:");
                print("X: $originalLeft px, Y: $originalTop px");
                print("Width: $originalWidth px, Height: $originalHeight px");
                // Call the cropImage function with the calculated crop parameters
                cropImage(originalLeft, originalTop, originalWidth, originalHeight);
              },
            ),
          Container(
            padding: const EdgeInsets.symmetric(horizontal: 10.0),
            child: DropdownButton<String>(
              value: selectedAspectRatio,
              items: ["1:1", "9:16", "16:9"].map((ratio) {
                return DropdownMenuItem(
                  value: ratio,
                  child: Text(ratio),
                );
              }).toList(),
              onChanged: (value) {
                setState(() {
                  selectedAspectRatio = value!;
                  cropRect = calculateCropRect(
                    constraints.maxWidth,
                    constraints.maxHeight,
                    widget.imageWidth!,
                    widget.imageHeight!,
                    selectedAspectRatio,
                  );
                });
              },
            ),
          ),
        ],
      ),
      body: Center(
        child: LayoutBuilder(
          builder: (context, constraints) {
            this.constraints = constraints;
            final maxWidth = constraints.maxWidth;
            final maxHeight = constraints.maxHeight;

            cropRect ??= calculateCropRect(
              maxWidth,
              maxHeight,
              widget.imageWidth!,
              widget.imageHeight!,
              selectedAspectRatio,
            );

            return Stack(
              children: [
                //InteractiveViewer restrict InteractiveViewer contain scale zoom in side CropRectBorderPainter
                Container(
                  color: Colors.black,
                  child: InteractiveViewer(
                    clipBehavior: Clip.hardEdge,
                    boundaryMargin: const EdgeInsets.all(0),
                    alignment: Alignment.center,
                    panEnabled: true, // Enable pan
                    transformationController: _transformationController,
                    minScale: 1.0,
                    maxScale: 4.0,
                    onInteractionUpdate: (details) {
                      setState(() {
                        zoomScale = details.scale;
                        panOffset = details.focalPoint; // Center point of the image as you interact
                      });
                    },
                    child: Image.file(
                      widget.file!,
                      fit: BoxFit.contain,
                      width: maxWidth,
                      height: maxHeight,
                    ),
                  ),
                ),
                if (cropRect != null)
                  Positioned.fill(
                    child: IgnorePointer(
                      child: Stack(
                        children: [
                          CustomPaint(
                            size: Size(maxWidth, maxHeight),
                            painter: BlackOverlayPainter(cropRect!),
                          ),
                          CustomPaint(
                            size: Size(maxWidth, maxHeight),
                            painter: CropRectBorderPainter(cropRect!),
                          )
                        ],
                      ),
                    ),
                  ),
              ],
            );
          },
        ),
      ),
    );
  }

  void _navigateToCroppedImageScreen(BuildContext context, String imagePath) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => CroppedImageScreen(imagePath: imagePath),
      ),
    );
  }

  // Function to crop the image using FFmpeg
  Future<void> cropImage(double x, double y, double width, double height) async {
    // Get the output directory
    Directory appDocDirectory = await getApplicationDocumentsDirectory();
    String outputPath = '${appDocDirectory.path}/cropped_image.jpg';

    // Ensure the paths are properly formatted for FFmpeg
    String inputFilePath = 'file://${widget.filePath}';
    String outputFilePath = 'file://$outputPath';
    // FFmpeg command to crop the image
    String command = '-y -i $inputFilePath -vf "crop=$width:$height:$x:$y" $outputFilePath';

    _croppedImagePath = outputPath; // Set the cropped image path

    FFmpegKitConfig.enableLogCallback((log) {
      final message = log.getMessage();
      print(message); // Enable logs to debug FFmpeg issues
    });

    // Execute the FFmpeg command to crop the image
    await FFmpegKit.execute(command).then((session) async {
      final returnCode = await session.getReturnCode();
      if (ReturnCode.isSuccess(returnCode)) {
        print('Image cropped successfully');
        print(_croppedImagePath); // Display cropped image path
        setState(() {
          // Update the UI with the cropped image path
          _navigateToCroppedImageScreen(context, _croppedImagePath);
        });
      } else {
        print('Error during cropping');
      }
    });
  }

  Rect calculateCropRect(double maxWidth, double maxHeight, double imgWidth, double imgHeight, String aspectRatio) {
    // Aspect ratio logic here
    double ratioWidth = 0.0;
    double ratioHeight = 0.0;

    switch (aspectRatio) {
      case "1:1":
        ratioWidth = maxWidth * 0.8; // Crop 80% of the screen width
        ratioHeight = ratioWidth; // Square aspect ratio
        break;
      case "9:16":
        ratioHeight = maxHeight * 0.8;
        ratioWidth = ratioHeight * (9 / 16); // 9:16 aspect ratio
        break;
      case "16:9":
        ratioWidth = maxWidth * 0.8;
        ratioHeight = ratioWidth / (16 / 9); // 16:9 aspect ratio
        break;
    }

    double offsetX = (maxWidth - ratioWidth) / 2;
    double offsetY = (maxHeight - ratioHeight) / 2;

    return Rect.fromLTWH(offsetX, offsetY, ratioWidth, ratioHeight);
  }
}

class BlackOverlayPainter extends CustomPainter {
  final Rect rect;

  BlackOverlayPainter(this.rect);

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()..color = Colors.black.withOpacity(0.5);
    canvas.drawRect(Offset(0, 0) & size, paint);
    paint.color = Colors.transparent;
    canvas.drawRect(rect, paint); // Make sure the crop area is transparent
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) {
    return false;
  }
}

class CropRectBorderPainter extends CustomPainter {
  final Rect rect;

  CropRectBorderPainter(this.rect);

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.white
      ..style = PaintingStyle.stroke
      ..strokeWidth = 2.0;
    canvas.drawRect(rect, paint);
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) {
    return false;
  }
}

class CroppedImageScreen extends StatelessWidget {
  final String imagePath;

  const CroppedImageScreen({Key? key, required this.imagePath}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Cropped Image')),
      body: Center(
        child: Image.file(File(imagePath)),
      ),
    );
  }
}
