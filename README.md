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

  
