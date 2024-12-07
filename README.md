# Mobile File Upload Demo

A cross-platform mobile file upload demo built with UniApp and Vue 3 Composition API.

## Features

- Cross-platform file selection (Android, iOS, H5)
- Native file picker integration
- Progress tracking for uploads
- Comprehensive error handling
- Platform-specific optimizations

## Setup

1. Clone the repository
2. Open the project in HBuilderX
3. Install dependencies (if any)
4. Run the project on your desired platform

## Platform Support

### Android
- Uses native Intent system for file picking
- Handles content:// URIs
- Supports various Android storage permissions

### iOS
- Uses UIDocumentPickerViewController
- Handles file system access
- iCloud document support

### H5
- Standard HTML5 file input
- Blob URL handling

## Usage

1. Click "选择文件" to open the file picker
2. Select your desired file
3. The selected file name will be displayed
4. Click "上传" to start the upload process
5. Progress will be shown during upload

## Notes

- Make sure to configure proper permissions in manifest.json
- Test thoroughly on different platforms
- Handle large files appropriately
