- The Camera System Will Contain 2 Classes
### 1) `PlayerCamera` Class:
- Methods:
	- Camera Shake Methods (Power/Duration ...) (See Addons)
	- Face Forward:
		- The Camera Will Face Forward The Player In the Facing Direction (Offset.x) + Rotation
	- Zoom In/Out (For Enemies Facing Or Cut-scenes)
	- focus on Object (Remote)
- Prpperties:
	- Offset
	- Smoothed
### 2) `CameraArea` Class:
- Camera Areas Limit The Camera and Add Bounds if player Enter it
	- Limits For Each Area In the Scene (Area Entered Limits Forced)

> [!NOTE] Connecting Method
> - All Connected By the `CameraManager` Auto-Loaded Class by signals
