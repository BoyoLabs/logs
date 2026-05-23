# Project Log: Webcam Utility

**Date:** 2026-05-20
**Objective:** Create a single-file HTML utility for webcam monitoring, photo capture, and video recording.

## Timeline of Events:

1. **Initial Implementation**:
   - Developed a single HTML file with CSS for styling and JavaScript for `getUserMedia` API integration.
   - Implemented mirrored video feed, canvas-based photo capture, and `MediaRecorder` for video clips.
   - Initial file saved to `/home/boyo/webcam-utility.html`.

2. **Deployment**:
   - Moved the utility to the Desktop (`/home/boyo/Desktop/webcam-utility.html`) per user request.

3. **Debugging - Visual Stream**:
   - **Issue**: User reported no visual stream after granting permission.
   - **Cause**: Browser autoplay policies often block unmuted video from starting automatically.
   - **Fix**: Added `muted` attribute to the `<video>` tag and added an explicit `await video.play()` call during initialization.

4. **Debugging - Audio Dependencies**:
   - **Issue**: Video stream failed to start because the audio source (microphone) could not be detected.
   - **Cause**: Requesting both video and audio fails entirely if either source is missing/blocked.
   - **Fix**: Implemented a fallback mechanism: the app now attempts to capture both audio and video, and if that fails, it tries again with video-only to ensure functionality.

## Final Result:
A robust, single-file HTML utility that successfully displays a live webcam feed and allows the user to save photos and video clips to their local machine, regardless of audio availability.
