# Note:
Currently on `V1.9.2` branch. To view other versions, please switch branches.

# How to get?

Please visit [https://www.insta360.com/sdk/apply](https://www.insta360.com/sdk/apply) to apply for the latest SDK.

# Support

Developers' Page: https://www.insta360.com/developer/home  
Insta360 Enterprise: https://www.insta360.com/enterprise  
Issue Report: https://insta.jinshuju.com/f/hZ4aMW  

# [中文文档](README_zh.md)

# Overview

The iOS SDK is mainly used to connect, set and obtain camera parameters, control the camera to take pictures and record, download files, upgrade firmware, and support video and image export, etc.

Supported models: X5, X4 Air, X4, X3, ONE X2, ONE X, ONE RS, ONE RS 1-Inch.

# Table of contents

* [INSCameraSDK](#inscamerasdk)
	* [Connection Module](#connection-module)
	* [Wi-Fi Control Module](#wi-fi-control-module)
	* [Information Retrieval Module](#information-retrieval-module)
	* [Notification Module](#notification-module)
	* [Sensor Device (Lens) Control Module](#sensor-device-lens-control-module)
	* [Preview Module](#preview-module)
	* [Capturing Control Module](#capturing-control-module)
	* [File Management Module](#file-management-module)
	* [CameraSDK Logger Module](#camerasdk-logger-module)
	* [GPS Data Module](#gps-data-module)
	* [Other Function Module](#other-function-module)
	* [Error Code](#error-code)
* [INSMediaSDK](#insmediasdk)
	* [Export Video](#export-video)
	* [Photo Export](#photo-export)
	* [Logging](#logging)
* [FAQ](#faq)



# Camera SDK Function

## Connection Module

### Determine if the Camera is Connected

Obtain the camera's connection status through the following methods:

```Swift
INSCameraManager.socket().cameraState
```

If the returned status is INSCameraStateConnected, it indicates a successful connection has been established. The status is defined as follows:

```C++
INSCameraManager.socket().cameraState

// There are the following states in total:
typedef NS_ENUM(NSUInteger, INSCameraState) {
    /// an insta360 camera is found, but not connected, will not response
    INSCameraStateFound,
    
    /// an insta360 camera is synchronized, but not connected, will not response
    INSCameraStateSynchronized,
    
    /// the nano device is connected, app is able to send requests
    INSCameraStateConnected,
    
    /// connect failed
    INSCameraStateConnectFailed,
    
    /// default state, no insta360 camera is found
    INSCameraStateNoConnection,
};
```

### Connecting the Camera

#### Connect the Camera via Wi-Fi

1. Connection

After connecting the camera to the phone via Wi-Fi, you can invoke the interface in INSCameraSDK to establish a connection. This interface uses the singleton pattern: an instance is created upon the first call, and subsequent calls reuse the same instance.

```C++
INSCameraManager.socket().setup()
```

2. Disconnection

When you need to disconnect from the camera, call the following interface:

```C++
INSCameraManager.socket().shutdown()
```

#### Connect the Camera via Bluetooth

> Note: Large data transfers are not supported when connected via Bluetooth. Therefore, functions involving the transfer of large data files are not supported.

Unsupported functions: Preview, retrieve supported list, download files, firmware upgrade, playback, export images.

1. Search for Bluetooth Devices

Call the following method to scan Bluetooth devices

```python
INSBluetoothManager().scanCameras
```

2. Establish a Bluetooth Connection

Establish a Bluetooth connection using the following interface:

```c++
- (id)connectDevice:(INSBluetoothDevice *)device
         completion:(void (^)(NSError * _Nullable))completion;
```

3. Disconnect Bluetooth

```Plain
- (void)disconnectDevice:(INSBluetoothDevice *)device;
```

### Send Heartbeat Packets to the Camera

While maintaining a connection with the camera, it is necessary to periodically send heartbeat packets to the camera. Otherwise, the camera will assume the connection has been lost, resulting in interrupted previews or invalid commands.

Typically, if no heartbeat is received within 30 seconds, the camera will actively disconnect.

Therefore, in scenarios requiring prolonged sessions—such as entering preview or recording modes—the application should continuously send heartbeat packets.

The following code example demonstrates how to use `GCDTimer` to send heartbeat commands at regular intervals, every 0.5 seconds:

```JavaScript
func startSendingHeartbeats() {
    let commandManager = INSCameraManager.shared().commandManager
    print("Heartbeat task started")
    
    // Send a heartbeat every 0.5 seconds to ensure the connection remains active.
    GCDTimer.shared.scheduledDispatchTimer(
        WithTimerName: "HeartbeatsTimer",
        timeInterval: 0.5,
        queue: DispatchQueue.main,
        repeats: true
    ) {
        commandManager.sendHeartbeats(with: nil)
        print("Heartbeat sent")
    }
}
```

## Wi-Fi Control Module

### Wi-Fi Information Retrieval

After establishing a Bluetooth connection, call the interface to retrieve Wi-Fi information. The sample code is as follows:

```javascript
let bluetoothManager = INSBluetoothManager()
bluetoothManager.command(by: peripheral)?.getOptionsWithTypes(optionTypes) { (err, options, successTypes) in
    guard let options = options else {
        self?.showAlert(row.tag!, String(describing: err))
        return
    }
    self?.showAlert("ssid:\(String(describing: options.wifiInfo?.ssid)) pw:\(String(describing: options.wifiInfo?.password))", String(describing: err))
}
```

### **Wi-Fi Country Codes and Camera Wi-Fi Channel Settings**

> ⚠️ Note: After setting the country code, you must restart the camera's Wi-Fi for the settings to take effect. The channel depends on the country code, with each country corresponding to a different channel list. After restarting the camera, retrieve the channel list again to switch channels (both Bluetooth and Wi-Fi connections are supported).

1. Retrieve the Current Country Code

```Plain
func getWifiCountryCode() {
    let optionTypes = [
        NSNumber(value: INSCameraOptionsType.wifiChannelList.rawValue)
    ]
    
    INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes) { err, options, successTypes in
        guard let options = options else {
            print("ERROR: \(err?.localizedDescription ?? "Unknown error")")
            return
        }
        self.printWifiChannelList(options.wifiChannelList)
    }
}
```

Note:

- Use `getOptionsWithTypes` to retrieve the camera's current Wi-Fi channel list and country code information.
- The callback returns `options.wifiChannelList`, which can be viewed using a custom print function.

2. Set the Country Code

```Plain
func setCountryCode(to countryCode: String = "JP") {
    let optionTypes = [
        NSNumber(value: INSCameraOptionsType.wifiChannelList.rawValue)
    ]
    
    let wifiChannelList = INSCameraWifiChannelList(countryCode: countryCode)
    let options = INSCameraOptions()
    options.wifiChannelList = wifiChannelList
    
    INSCameraManager.shared().commandManager.setOptions(options, forTypes: optionTypes) { error, types in
        if let error = error {
            print("Failed to set country code: \(error.localizedDescription)")
        } else {
            print("Country code set to \(countryCode). ⚠️ Wi-Fi restart required to apply changes.")
        }
    }
}
```

Note:

- The `countryCode` parameter supports any ISO country code, such as “JP”, ‘US’, or “CN”.
- After configuration, restart the camera's Wi-Fi for changes to take effect.
- The callback can be used to confirm whether the settings were applied successfully.

3. Restart Wi-Fi and Set the Channel.

```Plain
INSCameraManager.shared().commandManager.resetCameraWifi(channel)
```

## Information Retrieval Module

### Information Retrieval

#### Camera Models

```Plain
INSCameraManager.shared().currentCamera?.name
```

#### Camera Serial Number

The code can be called after establishing a connection with the camera

```swift
INSCameraManager.shared().currentCamera?.serialNumber
```

#### Firmware Version Number

```swift
INSCameraManager.shared().currentCamera?.firmwareRevision
```

#### Camera Activation Time

```swift
let optionTypes = [
         NSNumber(value: INSCameraOptionsType.activateTime.rawValue),
      ];
INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes) { (err, options, successTypes) in
    guard let options = options else {
        self.showAlert("get options", String(describing: err))
        return
    }
            // Activation time
    let formatter = DateFormatter()
    formatter.dateFormat = "yyyy-MM-dd HH:mm:ss.SSS"
    let date = Date(timeIntervalSince1970: Double(options.activateTime / 1000))
    print("date :\(date)")
    
}
```

#### SD Card Total Space (Total Capacity Size) and Free Space (Available Capacity Size)

```Python
@interface INSCameraStorageStatus : NSObject
// SD card status
@property (nonatomic) INSCameraCardState cardState;
// Available capacity size
@property (nonatomic) int64_t freeSpace;
// Total capacity size
@property (nonatomic) int64_t totalSpace;

// TODO: Restrict access to certain SDKs
@property (nonatomic) INSStorageCardLocation cardLocation;

@end
let optionTypes = [
            NSNumber(value: INSCameraOptionsType.storageState.rawValue),
            ];

INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes) { (err, options, successTypes) in
    guard let options = options else {
        self.showAlert("get options", String(describing: err))
        return
    }
}
```

#### SD Card Status

```swift
let optionTypes = [
      NSNumber(value: INSCameraOptionsType.storageState.rawValue),
      ];
INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes) { (err, options, successTypes) in
    guard let options = options else {
        self.showAlert("get options", String(describing: err))
        return
    }
    var sdCardStatus = "error"
    switch options.storageStatus?.cardState {
    case .normal:
        sdCardStatus = "Normal"
        break
    case .noCard:
        sdCardStatus = "NoCard"
        break
    case .noSpace:
        sdCardStatus = "NoSpace"
        break
    case .invalidFormat:
        sdCardStatus = "INvalid Format"
        break
    case .writeProtectCard:
        sdCardStatus = "Write Protect Card"
        break
    case .unknownError:
        sdCardStatus = "UnknownError"
        break
    default:
        sdCardStatus = "Status Error"
    }
}
```

#### Battery Level Information

```javascript
let optionTypes = [
         NSNumber(value: INSCameraOptionsType.batteryStatus.rawValue),
      ];
INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes) { (err, options, successTypes) in
    guard let options = options else {
        self.showAlert("get options", String(describing: err))
        return
    }
    print("battery: \(options.batteryStatus!.batteryLevel)")
}
```

#### Camera System Time

## Notification Module

### Battery Low Notification

**Notification string:** `INSCameraBatteryLowNotification`

```Swift
NotificationCenter.default.addObserver(self,
                                       selector: #selector(self.batteryLowNotification(_:)),
                                       name: NSNotification.Name.INSCameraBatteryLow,
                                       object: nil)
```

Callback example：

```Swift
@objc func batteryLowNotification(_ notification: Notification) {
    if let status = notification.userInfo?["storage_status"] as? INSCameraBatteryStatus {
        self.showAlert("Notification", "Battery Low: \(status.batteryScale)%")
    }
    print("Notification: Battery Low")
}
```

### SD Card Status Notification

**Notification string:** `INSCameraStorageStatusNotification`

```Objective-C
NotificationCenter.default.addObserver(self,
                                               selector: #selector(self.storageFullNotification(_:)),
                                               name: NSNotification.Name.INSCameraStorageStatus,
                                               object: nil)
```

Status types：

```Objective-C
typedef NS_ENUM(NSInteger, INSPBCardState) {
    INSPBCardState_GPBUnrecognizedEnumeratorValue = kGPBUnrecognizedEnumeratorValue,
    INSPBCardState_StorCsPass = 0,          // Normal
    INSPBCardState_StorCsNocard = 1,        // No card
    INSPBCardState_StorCsNospace = 2,       // No space
    INSPBCardState_StorCsInvalidFormat = 3, // Invalid format
    INSPBCardState_StorCsWpcard = 4,        // Write protection
    INSPBCardState_StorCsOtherError = 5     // Other error
};
```

Callback example：

```Swift
@objc func storageFullNotification(_ notification: Notification) {
    
    if let status = notification.userInfo?["card_state"] as? INSCameraCardState {
        self.showAlert("Notification", "Battery Low :\(status)")
    }
    
    print("Notification: Storage Full")
}
```

### Camera Capturing Stopped Notification (Anomaly)

**Notification string:** `INSCameraCaptureStoppedNotification`

```Objective-C
NotificationCenter.default.addObserver(self,
                                        selector: #selector(self.captureStoppedNotification(_:)),
                                        name: NSNotification.Name.INSCameraCaptureStopped,
                                        object: nil)
```

Callback example：

```Swift
@objc func captureStoppedNotification(_ notification: Notification) {
    
    if let info = notification.userInfo?["video"] as? INSCameraVideoInfo, let errorCode = notification.userInfo?["error_code"] as? Int{
        self.showAlert("Notification", "Capture Stop, error code:\(errorCode)")
    }
    
    print("Notification: Capture Stop")
}
```

> **Note:** If any anomalies occur during recording, such as storage errors or excessive temperature, the device will automatically stop recording. In such cases, the user should be prompted to restart the operation to prevent loss of important content.

### CameraTemperature Status Notification (Overheating)

**Notification string:** `INSCameraTemperatureStatusNotification`

```Objective-C
NotificationCenter.default.addObserver(self,
                                        selector: #selector(self.temperatureValueNotification(_:)),
                                        name: NSNotification.Name.INSCameraTemperatureStatus,
                                        object: nil)
```

Callback example：

```Swift
@objc func temperatureValueNotification(_ notification: Notification) {
    
    if let status = notification.userInfo?["temperature_status"] as? INSCameraTemperatureStatus {
        self.showAlert("Notification", "Temperature warning, \(status.temperature) ")
    }
    
    print("Notification: Temperature warning")
}
```

> **Note:** High temperatures may cause reduced performance or even damage to the camera. Upon receiving this notification, users should be advised to cease operation and allow the camera to cool before resuming use.

## Sensor Device (Lens) Control Module

The following is the enumeration of sensor device types (lens types) available for the camera:

```Plain
typedef NS_ENUM(NSUInteger, INSSensorDevice) {
    INSSensorDeviceUnknown, // Unknown
    INSSensorDeviceFront,   // Front
    INSSensorDeviceRear,    // Rear
    INSSensorDeviceAll,     // Panoramic
};
```

### Retrieve Active Sensor Type

```Go
INSCameraManager.socket().commandManager.getActiveSensor { error, device, str1, str2 in
    if error != nil {
        self.showAlert("HDR Take:", "Failed!")
        return
    }
    
    print("device:\(device)")
}
```

- For cameras with interchangeable lenses (One RS, One R), use the following code to retrieve the current sensor type: Wide indicates wide-angle, Pano indicates panorama:

```Swift
func getLensType(){
    let settings: INSCameraDeviceSettings? = INSCameraManager.shared().currentCamera?.settings
    guard let mediaOffset = settings?.mediaOffset else {
        return
    }
    
    let secnsorType = INSLensOffset(offset: mediaOffset).lensType
    if secnsorType == INSLensType.oneR577Wide.rawValue {
        print("secnsorType:577Wide")
    } else if secnsorType == INSLensType.oneR577Pano.rawValue{
        print("secnsorType:577Pano")
    }
    
    print("secnsorType:\(secnsorType)")
}
```

### Switching Sensors

```Plain
INSCameraManager.shared().commandManager.setActiveSensorWith(activeSensorDevice) { [weak self] err, mediaOffset, mediaOffsetV3 in
    if let err = err {
        self?.showAlert("失败", String(describing: err))
    } else {
        self?.showAlert("成功", "current offset: \(mediaOffset ?? "")")
        self?.shouldRestartPreview = true
        self?.runMediaSession()
    }
}
```

Usage Notes:

- `activeSensorDevice`: Pass the sensors type to switch to (e.g., .front, .rear, .all)

## Preview Module

The preview has the following core classes; for details, refer to:`demo：CameraConfigByJsonController`

- `INSCameraSessionPlayer`
  - Bind iOS-side `bounds`
  - Implementation protocol: `INSCameraSessionPlayerDelegate`
  - Implementation protocol: `INSCameraSessionPlayerDataSource`, it is recommended to implement the following two protocols:
    1. Configure Offset information: `updateOffsetToPlayer`
    2. Configure stabilization information: `updateStabilizerParamToPlayer`
- `INSCameraMediaSession`

```swift
    func setupRenderView() {
        var frame = self.view.bounds;
        let height = frame.size.width * 0.667
        frame.origin.x = 0
        frame.origin.y = frame.size.height - height
        frame.size.height = height
        
        newPlayer = INSCameraSessionPlayer()
        newPlayer?.delegate = self
        newPlayer?.dataSource = self
        newPlayer?.render.renderModelType.displayType = .sphereStitch
        
        if let view = newPlayer?.renderView {
            view.frame = frame;
            print("预览流: getView = \(view), frame = \(frame)")
            self.view.addSubview(view)
        }
        
        let config = INSH264DecoderConfig()
        config.shouldReloadDecoder = false
        config.decodeErrorMaxCount = 30
        config.debugLiveStrem = false
        config.debugLog = false
        self.mediaSession.setDecoderConfig(config)
    }
```

### Preview Real-Time Stabilization Data Acquisition

During the preview process, implementing the `INSCameraSessionGyroDelegate` protocol in `INSCameraSessionPlayer` enables real-time callbacks for stabilization data. The callback data exists in two formats: raw `NSData` and parsed `INSGyroRawItem` format.

```objective-c
@protocol INSCameraSessionGyroDelegate <NSObject>

@optional

- (void)onRawGyroData:(NSData *)rawData timestampMs:(int64_t)timestampMs;

- (void)onParsedGyroData:(NSMutableArray<INSGyroRawItem *> *)gyroItems timestampMs:(int64_t)timestampMs;

@end
@interface INSCameraSessionPlayer : NSObject<INSCameraPlayerSession>

@property (nonatomic, weak  ) id<INSCameraSessionGyroDelegate> gyroDelegate;

@end
```

## Capturing Control Module

This module primarily covers camera mode switching, parameter configuration, and functions for retrieving the camera's current capturing mode and settings. The **JSON configuration mechanism** is a key focus.

The camera interface supports configuring all parameters, but note that not all parameters take effect in every mode. For example, setting the photo interval parameter in video recording mode is ineffective. To address this, the camera internally maintains a **JSON configuration table** defining the list of available parameters and their valid ranges for each mode.

Developers can download this JSON configuration table to accurately retrieve the available parameters and their corresponding values for different capture modes, enabling more rational and efficient camera control and configuration.

Additionally, we provide a management class, `INSCameraAttrManagerWrapper`, to maintain camera parameters and their dependencies. This class stores all camera parameters and determines parameter support and valid ranges based on the JSON configuration table. Refer to **CameraConfigByJsonController.swift** in the demo for usage details.

In practice, developers can first retrieve current parameters from the camera and set them in this class. Class methods can then be used to query parameter support status and available values.

### Retrieve the Camera JSON Configuration File

For X4 and later cameras, capture modes and attributes are determined by the properties configured in the supported list. Different cameras support varying capture modes, and supported attributes also differ across modes. Before capturing, synchronize the camera's supported list (JSON file).

```Objective-C
INSCameraManager.socket().commandManager.fetchStorageFileInfo(with: .json, completion: {error, fileResp in
    if let err = error {
        return
    }
    
    guard let fileResp = fileResp else {
        return
    }
    
    print("Info: fetchStorageFileInfo Success!")
    
    INSCameraManager.socket().commandManager.fetchResource(withURI: fileResp.uri, toLocalFile: outputURL, progress: {progress in
    }, completion: { error in
        
        if let error = error {
            print("Error: fetchResource Failed!")
            return
        }
        
        print("Info: fetchResource Success!")
        
        completion?()
    })
    
})
```

### Retrieve the Current Capture Mode

Use the interface `INSCameraManager.shared().commandManager.getOptionsWithTypes` to retrieve the current camera's capture mode. This method requires the following parameters:

- **optionTypes** Specify which parameters to retrieve. Each parameter corresponds to an enumeration value (see INSCameraOptionsType) and must be converted to an `NSNumber` array.
- **requestOptions** The maximum configurable request timeout duration (default 10 seconds).
- **completion** Returned in callback:
  - `error`: `nil` indicates successful retrieval; otherwise, contains the error reason.
  - `cameraOptions`: Contains the current camera capture mode (capture mode is either photoSubMode or videoSubMode; see below for specific enumeration meanings).

Capture mode enumeration：

```typescript
// Photo capturing
typedef NS_ENUM(NSUInteger, INSPhotoSubMode) {
    INSPhotoSubModeSingle = 0,
    INSPhotoSubModeHdr,
    INSPhotoSubModeInterval,
    INSPhotoSubModeBurst,
    INSPhotoSubModeNightscape,
    INSPhotoSubModeInstaPano,
    INSPhotoSubModeInstaPanoHdr,
    INSPhotoSubModeStarlapse,
    INSPhotoSubModeNone = 100,
};
// Video recording
typedef NS_ENUM(NSUInteger, INSVideoSubMode) {
    INSVideoSubModeNormal = 0,
    INSVideoSubModeBulletTime,
    INSVideoSubModeTimelapse,
    INSVideoSubModeHDR,
    INSVideoSubModeTimeShift,
    INSVideoSubModeSuperVideo,
    INSVideoSubModeLoopRecording,
    INSVideoSubModeFPV,
    INSVideoSubModeMovieRecording,
    INSVideoSubModeSlowMotion,
    INSVideoSubModeSelfie,
    INSVideoSubModePure,
    INSVideoSubModeLiveview,
    INSVideoSubModeCameraLiveview,
    INSVideoSubModeDashCam,
    INSVideoSubModePTZ,
    INSVideoSubModeNone = 100,
};
```

Call demo:

```typescript
func getCurrentCameraMode(completion: (()->Void)?)->Void{
    
    let typeArray: [INSCameraOptionsType] = [.videoSubMode, .photoSubMode]
    
    let optionTypes = typeArray.map { (type) -> NSNumber in
        return NSNumber(value: type.rawValue)
    }
    let requestOptions = INSCameraRequestOptions()
    requestOptions.timeout = 10
    
    INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes, requestOptions: requestOptions) { [weak self] error, cameraOptions, _ in
        if let error = error {
            self?.showAlert("Failed", "The camera mode has been get failed: \(error.localizedDescription)")
            return
        }
        
        guard let cameraOptions = cameraOptions else {return}
        var captureMode: HUMCaptureMode? = nil
        if cameraOptions.photoSubMode != 100 {
            captureMode = HUMCaptureMode.modePhotoFrom(value: cameraOptions.photoSubMode)
        }
        else if cameraOptions.videoSubMode != 100 {
            captureMode = HUMCaptureMode.modeVideoFrom(value: cameraOptions.videoSubMode)
        }
        guard let mode = captureMode else {return}
        completion?()
    }
}
```

### Switching Capture Mode

The method `INSCameraManager.shared().commandManager.setOptions` configures the specified mode for the camera. When setting the capture mode, note the following key points:

- **INSCameraOptions**

Holds the capture mode configuration, primarily comprising two properties:

- `photoSubMode`: Photo mode
- `videoSubMode`: Video mode

- **INSCameraOptionsType**

Specifies which parameters in `INSCameraOptions` should take effect:

- Photo mode: `INSCameraOptionsType::INSCameraOptionsTypePhotoSubMode`
- Video mode: `INSCameraOptionsType::INSCameraOptionsTypeVideoSubMode`

The following is sample code (using photo mode configuration as an example):

```C++
// Photo Capture Mode Enumeration
typedef NS_ENUM(NSUInteger, INSPhotoSubMode) {
    INSPhotoSubModeSingle = 0,
    INSPhotoSubModeHdr,
    INSPhotoSubModeInterval,
    INSPhotoSubModeBurst,
    INSPhotoSubModeNightscape,
    INSPhotoSubModeInstaPano,
    INSPhotoSubModeInstaPanoHdr,
    INSPhotoSubModeStarlapse,
    INSPhotoSubModeNone = 100,
};

var typeArray: [INSCameraOptionsType] = [.photoSubMode]
let optionTypes = typeArray.map { NSNumber(value: $0.rawValue) }
let cameraOptions = INSCameraOptions()
cameraOptions.photoSubMode = INSPhotoSubMode.single

INSCameraManager.shared().commandManager.setOptions(cameraOptions, forTypes: optionTypes) { [weak self] error, _ in
    if let error = error {
        self?.showAlert("Failed", "The camera mode has been set failed: \(error.localizedDescription)")
        print("Failed: The camera mode has been set failed: \(error.localizedDescription)")
        return
    }
    print("SUCCESS")
}
```

### Parameter Setting

#### Capturing Parameters

##### White Balance (Numeric Type)

The property in `INSPhotographyOptions` is:

```Objective-C
@property (nonatomic) uint32_t whiteBalanceValue;
```

The corresponding enumeration in `INSPhotographyOptionsType` is:

```Objective-C
INSPhotographyOptionsTypeWhiteBalance = 13
```

Assuming the current shooting mode `INSCameraFunctionMode` is 6, indicating standard photo capture.

- X2 and One X only support enumeration (`INSCameraWhiteBalance`) settings; X3 and later models support using `whiteBalanceValue`.

```Swift
@property (nonatomic) uint32_t whiteBalanceValue;
@property (nonatomic) INSCameraWhiteBalance whiteBalance;
typedef NS_ENUM(uint16_t, INSCameraWhiteBalance) {
    INSCameraWhiteBalanceAutomatic = 0,
    
    /// Inscandescent
    INSCameraWhiteBalanceColorTemp2700K,
    
    /// Sunny.   ONE X Fluorescent
    INSCameraWhiteBalanceColorTemp4000K,
    
    /// Cloudy.  ONE X Sunny
    INSCameraWhiteBalanceColorTemp5000K,
    
    /// Fluorescent. ONE X Cloudy
    INSCameraWhiteBalanceColorTemp6500K,
    
    /// Outdoor.
    INSCameraWhiteBalanceColorTemp7500K = 5,
};
```

##### ExposureBias（Numeric Type）

Adjustment is only supported when the exposure mode is set to “AUTO” or “FULL_AUTO”.

The property in `INSPhotographyOptions` is: `exposureBias`

```Objective-C
@property (nonatomic) float exposureBias;
```

The corresponding enumeration in `INSPhotographyOptionsType` is:

```Objective-C
INSPhotographyOptionsTypeExposureBias = 7
```

##### ISO，shutterSpeed（Numeric Type）

Adjustment is supported only in the “MANUAL” and “ISO_PRIORITY” exposure modes.

The respective properties in `INSPhotographyOptions` are: `stillExposure.iso` `stillExposure.shutterSpeed`

```objective-c
@property (nullable, nonatomic) INSCameraExposureOptions *stillExposure;


@interface INSCameraExposureOptions : NSObject

@property (nonatomic) uint8_t program;

@property (nonatomic) NSUInteger iso;

#if TARGET_OS_IOS
@property (nonatomic) CMTime shutterSpeed;
#endif

@end
```

The corresponding enumeration in `INSPhotographyOptionsType` is:

```Objective-C
INSPhotographyOptionsTypeStillExposureOptions = 20
```

##### lapse_time（Numeric Type）

This parameter is only effective in `timelapse` mode.

```JavaScript
    func setLapseTimeToCamera(completion: (()->Void)?){
        
        let functionMode = self.attrManage.getIntFunctionsMode()
        let functionModeHum = HUMCaptureMode.modeFuncionFrom(value: functionMode)
        
        var mode:INSTimelapseMode? = nil
        
        if functionModeHum.isPhotoMode {
            mode = INSTimelapseMode.image
        }else if functionModeHum.isVideoMode {
            mode = INSTimelapseMode.video
        }
        
        guard let mode = mode else {return}
        
        guard let options = self.currentLapseTime else {return}
        
        options.lapseTime = UInt(self.attrManage.getLapseTime() * 1000)
//        options.duration  = self.attrManage.getRecordDuration()
        INSCameraManager.shared().commandManager.setTimelapseOptions(options, for: mode, completion: { (err) in
            if let err = err {
                self.showAlert("Set LapseTime failed", err.localizedDescription);
                return
            }
            completion?()
        })
        
    }
```

##### Resolution（Enumeration）

Starting from version `1.8.1`, resolution is uniformly managed via enumeration. You must manually configure `isUseJsonOptions` to `true`.

```Objective-C
INSPhotographyOptions.isUseJsonOptions = true
```

- Photo

The property in `INSPhotographyOptions` is: `photoSizeForJson`

```
photoSizeForJson = 17`, indicating that the x4 camera has a resolution of `8192 x 6144
```

The corresponding enumeration in `INSPhotographyOptionsType` is:

```Objective-C
INSPhotographyOptionsTypePhotoSize = 40
```

Assuming the current shooting mode `INSCameraFunctionMode` is 6, indicating standard photo capture.

Code example：

```swift
let functionMode = INSCameraFunctionMode.init(functionMode: 6)
var typeArray: [INSPhotographyOptionsType] = [.photoSize]
let optionTypes = typeArray.map { NSNumber(value: $0.rawValue) }

options.isUseJsonOptions = true
option.photoSizeForJson = 17

INSCameraManager.shared().commandManager.setPhotographyOptions(option, for: functionMode, types: optionTypes, completion:  { [weak self] error, _ in
    guard let error = error else {
        self?.showAlert("Success", " set successfully")
        return
    }
    self?.showAlert("Failed", "set failed: \(error.localizedDescription)")
})
```

Correspondence between Resolution and Enumeration Values

```Swift
"photo_resolution": {
    "6912_3456": "0",  // X3 18MP
    "6272_3136": "1",
    "6080_3040": "2",
    "4000_3000": "3",
    "4000_2250": "4",
    "5212_3542": "5",
    "5312_2988": "6",
    "8000_6000": "7",
    "8000_4500": "8",
    "2976_2976": "9",
    "5984_5984": "10",
    "11968_5984": "11",
    "5952_2976": "12",  // 72MP
    "8192_6144": "17",  // X4 18MP
    "8192_4608": "18",
    "4096_3072": "19",
    "4096_2304": "20",
    "7680_3840": "21",
    "3840_3840": "22"
}
```

- Video

The property in `INSPhotographyOptions` is: `videoResolutionForJson`

Not displayed here; refer to the `record_resolution` list in the `common_camera_setting_proto.json` file.

##### Exposure Mode (Enumeration)

The properties in `INSPhotographyOptions` are:

```Objective-C
@property (nullable, nonatomic) INSCameraExposureOptions *stillExposure;


@interface INSCameraExposureOptions : NSObject

@property (nonatomic) uint8_t program; // Exposure mode

@property (nonatomic) NSUInteger iso;

#if TARGET_OS_IOS
@property (nonatomic) CMTime shutterSpeed;
#endif

@end
```

The corresponding enumeration in `INSPhotographyOptionsType` is:

```Objective-C
INSPhotographyOptionsTypeStillExposureOptions = 20
"exposure_program": {
   "AUTO": "0",
   "ISO_PRIORITY": "1",
   "SHUTTER_PRIORITY": "2",
   "MANUAL": "3",
   "ADAPTIVE": "4",
   "FULL_AUTO": "5"
},
```

Supports setting multiple parameters for the camera simultaneously, as follows:

```Swift
var typeArray: [INSPhotographyOptionsType] = [.photoSize, .whiteBalanceValue]
let optionTypes = typeArray.map { NSNumber(value: $0.rawValue) }

option.photoSizeForJson = 11
option.whiteBalanceValue = 4000

INSCameraManager.shared().commandManager.setPhotographyOptions(option, for: functionMode, types: optionTypes, completion:  { [weak self] error, _ in
    guard let error = error else {
        self?.showAlert("Success", "The camera resolution has been set successfully")
        return
    }
    self?.showAlert("Failed", "The camera resolution has been set failed: \(error.localizedDescription)")
    
    completion?()
})
```

##### Capture Type (Enumeration)

Note: The X5 defaults to PureShot mode and does not support disabling it.

The property in `INSPhotographyOptions` is:

```Objective-C
@property (nonatomic) uint8_t rawCaptureType;
```

The corresponding enumeration in `INSPhotographyOptionsType` is:

```Objective-C
INSPhotographyOptionsTypeRawCaptureType = 25,
```

Enumeration type:

```json
    "raw_capture_type": {
       "OFF": "0",
       "RAW": "1",
       "PURESHOT": "3",
       "PURESHOT_RAW": "4",
       "INSP": "5",
       "INSP_RAW": "6"
    },
```

##### In-camera Splicing (Bool)

When capturing photos, the camera internally splices panoramic images, eliminating the need for external splicing of source material. Supported only on x4 and later camera models.

```swift
var typeArray = [INSCameraOptionsType]()
let cameraOptions = INSCameraOptions()

// Open is true, Closed is false
cameraOptions.enableInternalSplicing = true
typeArray.append(.internalSplicing)

let optionTypes = typeArray.map { (type) -> NSNumber in
    return NSNumber(value: type.rawValue)
}
INSCameraManager.shared().commandManager.setOptions(cameraOptions, forTypes: optionTypes) { [weak self] error, _ in
    if let error = error {
        self?.showAlert("Failed", "The camera mode has been set failed: \(error.localizedDescription)")
        return
    }
    completion?()
}
```

#### Retrieve Current Capture Parameters

The camera is obtained via the `getPhotographyOptions` method. The primary input parameters required are three: `functionMode` and `optionTypes`, along with a closure callback.

```swift
/*!
 * @discussion  Get photography options. Your app should not call this method to get the current photography options. Instead, your app should rely on the photography options that just set, or they will be as default.
 *
 * @param   functionMode the target function mode to get
 * @param   optionTypes array of the INSPhotographyOptionsType to get
 * @param   completion  the callback block. If all succeed, error would be nil, if partial failed, error's code would be <code>INSCameraErrorCodeAccept</code>, if all fail, error's code would be the maximum error code.
 */
- (void)getPhotographyOptionsWithFunctionMode:(INSCameraFunctionMode*)functionMode
                                        types:(NSArray <NSNumber *>*)optionTypes
                                   completion:(void(^)(NSError * _Nullable error, 
                                   INSPhotographyOptions * _Nullable options, 
                                   NSArray <NSNumber *>* _Nullable successTypes))completion;
```

- `functionMode`: Indicates the current shooting mode
- `optionTypes`: Each parameter corresponds to an enumeration value
  -  For example, 

  - Resolution corresponds to:`INSPhotographyOptionsTypePhotoSize = 40`
  - White balance corresponds to: `INSPhotographyOptionsTypeWhiteBalance = 13`
- The closure returns an `NSError` and `INSPhotographyOptions`
  - `NSError` being `nil` indicates success; otherwise, it contains the reason for failure
  - `INSPhotographyOptions` contains the specific values of current camera parameters, such as resolution, white balance, etc.

(Example) Retrieve the current camera white balance value:

```swift
var typeArray: [INSPhotographyOptionsType] = [.whiteBalanceValue]
let optionTypes = typeArray.map { NSNumber(value: $0.rawValue) }

let functionMode = currentFunctionMode.functionMode

INSCameraManager.shared().commandManager.getPhotographyOptions(with: functionMode, types: optionTypes, completion: { [weak self] error, camerOptions, successTags inif let error = error {print("Get Message Failed! Error: \(error.localizedDescription)")return
    }
    print("getPhotographyOptions Success!")
    print("whiteBalance: \(camerOptions?.whiteBalance ?? "N/A")"）
})
```

#### Setting Camera Parameters

To configure camera parameters, use the method: `INSCameraManager.shared().commandManager.setPhotographyOptions`. 

The primary input parameters are as follows:

- `INSPhotographyOptions`: Stores the configured parameters
- `INSPhotographyOptionsType`: Indicates which parameters require configuration (each parameter corresponds to an enumeration)
- `INSCameraFunctionMode`: The current capture mode.
  - 相机设置需要的拍摄模式`INSCameraFunctionMode`和从相机拿到的拍摄模式`INSPhotoSubMode`或`INSVideoSubMode`不能直接转换，对应关系参考工程文件中的类：`HUMCaptureMode`。通过`HUMCaptureMode`将相机当前模式`INSPhotoSubMode`或`INSVideoSubMode`转成统一的`INSCameraFunctionMode`
  - The camera settings require the `INSCameraFunctionMode`, which cannot be directly converted from the `INSPhotoSubMode` or `INSVideoSubMode` obtained from the camera. It can refer to the class `HUMCaptureMode` in the project documentation for the mapping relationship. Use `HUMCaptureMode` to convert the camera's current mode (INSPhotoSubMode or INSVideoSubMode) into the unified INSCameraFunctionMode.
- Closure callback: Returns whether the retrieval was successful.

All parameter types must be set to values supported by the camera screen and cannot be set arbitrarily.

Sample code is as follows: Set specified white balance

```Swift
let functionMode = INSCameraFunctionMode.init(functionMode: 6)
var typeArray: [INSPhotographyOptionsType] = [.whiteBalanceValue]
let optionTypes = typeArray.map { NSNumber(value: $0.rawValue) }

option.whiteBalanceValue = 4000
INSCameraManager.shared().commandManager.setPhotographyOptions(option, for: functionMode, types: optionTypes, completion:  { [weak self] error, _ in
    guard let error = error else {
        self?.showAlert("Success", "set successfully")
        return
    }
    self?.showAlert("Failed", "set failed: \(error.localizedDescription)")
})
```

### Photo Capturing

#### Standard Capturing

通过`INSCameraManager.shared().commandManager.takePicture`方法向相机下达拍摄指令，若 `error` 为 `nil`，则表示拍摄成功，反之则包含错误信息。`optionInfo` 中涵盖照片的远程访问地址，可进行远程导出（无需下载至本地）。 

The camera is commanded to take a photo via the `INSCameraManager.shared().commandManager.takePicture` method. If `error` is `nil`, the photo capture was successful; otherwise, it contains error information. The `optionInfo` contains the remote access URL for the photo, enabling remote export (without downloading to the local device). 

```bash
INSCameraManager.shared().commandManager.takePicture(with: nil, completion: { (error, optionInfo) in
    
    print("end takePicture \(Date())")     
    guard let uri = optionInfo?.uri else {
        return
    }
    print("Take Picture Url:\(uri)")\
})
```

#### HDR Photo Capturing

Operation Method Change (x4 and Later Models):

For x4 and later models, switch to the corresponding capturing mode before initiating the shot.

```Swift
let options:INSTakePictureOptions = INSTakePictureOptions()
INSCameraManager.shared().commandManager.takePicture(with: options, completion: { (error, optionInfo) in
  guard let uri = optionInfo?.uri else {
        return
  }
})
```

For HDR capturing on the X3 and earlier models, you need to set the mode to `AEB` mode in capturing parameters before taking the photo.

```Swift
let options:INSTakePictureOptions = INSTakePictureOptions()
options.mode = INSPhotoMode.aeb

INSCameraManager.shared().commandManager.takePicture(with: options, completion: { (error, optionInfo) in
    guard let uri = optionInfo?.uri else {
        return
    }
})
```

#### Countdown  photo capture (only supports X5 and X4 Air)

You need to set the `photographySelfTimer` parameter using `setPhotographyOptions`, and set `countDown` to 0.

### Video Recording

#### Standard Recording

开始录制：

```python
INSCameraManager.shared().commandManager.startCapture(with: nil, completion: { (err) in
    if let err = err {
        self.showAlert(row.tag!, "\(String(describing: err))")
        return
    }
})
```

Stop recording

```python
INSCameraManager.shared().commandManager.stopCapture(with: nil, completion: { (err, info) in
    if let err = err {
        self.showAlert(row.tag!, "\(String(describing: err))")
        return
    }
})
```

#### Timelapse Recording

Starting and stopping recording follows the standard procedure, but there are some important points to note:

- For the X5 and later models, if you need to save IMU data during timelapse recording, you must use our SDK for recording. Using our app for recording will not save IMU data.
- Furthermore, the IMU data storage function must be triggered by calling the following command:

Start recording

```Objective-C
/*!
 * Start time lapse capture. For ONE X, you can input `INSExtraInfo` via options if the `INSTimelapseMode` is image.
 *
 * availability(ONE, ONE X)
 */
- (void)startCaptureTimelapseWithOptions:(INSStartCaptureTimelapseOptions  * _Nonnull)options completion:(void(^)(NSError * _Nullable error))completion;
```

Stop recording

```Objective-C
/*!
 * Start time lapse capture
 *
 * availability(ONE)
 */
- (void)stopCaptureTimelapseWithCompletion:(void(^)(NSError * _Nullable error,
                                                    INSCameraVideoInfo * _Nullable videoInfo))completion;
```

## File Management Module

### Get File List

```Plain
- (void)fetchPhotoListWithOptions:(INSGetFileListOptions *)options
                       completion:(void (^)(NSError * _Nullable error, INSCameraResources * _Nullable res))completion;
```

### File Download

```Plain
- (NSURLSessionTask *)fetchResourceWithURI:(NSString *)URI 
                               toLocalFile:(NSURL *)localFileURL
                                  progress:(void (^)(NSProgress * _Nullable progress))progress
                                completion:(void(^)(NSError * _Nullable error))completion;
```

## CameraSDK Logger Module

In `INSCameraSDK`, all log information is passed to the upper layer as callbacks via the `INSCameraSDKLoggerProtocol` protocol. Upon receiving these logs, the upper layer can choose to print them for developers to observe the SDK's operational status during debugging.

Logs can also be stored persistently to support subsequent data analysis or troubleshooting. For detailed configuration methods, refer to the sample controller `CameraConfigByJsonController`.

Log callbacks are implemented via the INSCameraSDKLogger singleton object, which serves as the protocol delegate for `INSCameraSDKLoggerProtocol`.

Developers must assign their custom log delegate object to the `logDelegate` property of `INSCameraSDKLogger` to receive log messages of all levels output by the SDK.

The logging protocol `INSCameraSDKLoggerProtocol` includes the following methods:

```objective-c
@protocol INSCameraSDKLoggerProtocol

- (void)logError:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logWarning:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logInfo:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logDebug:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logCrash:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;

@end
```

## GPS Data Module

### Settings in Photo Capture Mode

When calling the capture interface, pass the `INSMediaGps` data into the capture parameters.

```JavaScript
func pictureGps() {
    // Constructing current location (GPS) data, using Hong Kong coordinate points as an example.
    let newestLocation = CLLocation(
        coordinate: CLLocationCoordinate2D(latitude: 22.2803, longitude: 114.1655), // 经纬度
        altitude: 10.0,              // Altitude (meters)
        horizontalAccuracy: 5.0,     // Horizontal Accuracy (meters)
        verticalAccuracy: 5.0,       // Vertical Accuracy (meters)
        timestamp: Date()            // Current Timestamp
    )

    // Create extended metadata objects (for attaching GPS, gyroscope, magnetometer, and other information)
    let metaData = INSExtraMetadata()

    // Create a media GPS information object using a constructed CLLocation.
    let mediaGps = INSMediaGps(clLocation: newestLocation, isValidLocation: true)

    // Write GPS information into the metadata
    metaData.gps = mediaGps

    // Create an extended information object and set its metadata (gyroscope data is not set here).
    let extraInfo = INSExtraInfo(
        version: Int32(INSExtraInfoVersion.one2.rawValue),  // Using Extended Information Version one2
        metadata: metaData,                                 // Metadata with GPS information
        gyroData: nil                                       // No gyroscope data
    )

    // Construct a photo selection object containing extended information
    let options: INSTakePictureOptions = INSTakePictureOptions(extraInfo: extraInfo)

    // Call the camera to take a photo
    INSCameraManager.shared().commandManager.takePicture(with: options, completion: { (error, optionInfo) in
        print("end takePicture \(Date())") // Printing the photo capture completion time

        // Extract the photo URI from the response data
        guard let uri = optionInfo?.uri else {
            return // If retrieval fails, return directly.
        }

        print("Take Picture Url:\(uri)") // Print the Url of the photo file
    })
}
```

### Settings in Video Recording Mode

After starting recording, transmit the constructed GPS data to the camera via `uploadGpsDatas`.

```JavaScript
// Create an extra information object (typically used to pass extended parameters)
let extraInfo = INSExtraInfo()

// Create a recording parameter object and set extraInfo.
let options = INSCaptureOptions(extraInfo: extraInfo)

// Start capturing
self?.commandsManager.startCapture(with: options, completion: { (err) in
    if let err = err {
        // Startup failed, pop-up alert
        print("\(row.title!) failed with error: \(err)")
        self?.showAlert(row.title!, err.localizedDescription)
        return
    }

    // Execute GPS data upload after a 5-second delay.
    DispatchQueue.main.asyncAfter(deadline: .now() + 5) {

        // Create a GPS data array (simulating 100 entries)
        var gpsDatas: [INSCameraGpsInfo] = []
        for _ in 0..<100 {
            let location = CLLocation(
                coordinate: CLLocationCoordinate2DMake(22.219749, 11.264474),
                altitude: 50.0,
                horizontalAccuracy: 1.0,
                verticalAccuracy: 1.0,
                course: 0.012,
                speed: 10.0,
                timestamp: Date()
            )
            let mediaGps = INSCameraGpsInfo(clLocation: location, isValidLocation: true)
            gpsDatas.append(mediaGps)
        }

        // Convert to binary data format (optional: for debugging or uploading)
        var datas = Data()
        for gps in gpsDatas {
            guard let gpsD = gps.toGpsData() else {
                continue
            }
            datas.append(gpsD)
        }
        print("--- \(datas)") // Output debugging information

        // Upload GPS data to the camera
        INSCameraManager.shared().commandManager.uploadGpsDatas(gpsDatas, completion: { (error) in
            if let error = error {
                self?.showAlert("error", "upload gps datas error: \(error)")
                print("upload gps datas error: \(error)")
            } else {
                self?.showAlert("success", "upload gps datas success")
                print("upload gps datas success")
            }

            // Stop recording after upload completes.
            INSCameraManager.shared().commandManager.stopCapture(with: options, completion: { [weak self] (err, video) in
                if let err = err {
                    print("\(row.title!) failed with error: \(err)")
                    self?.showAlert(row.title!, err.localizedDescription)
                    return
                }
            })
        })
    }
})
```

## Other Function Module

### Camera Activation

The App ID and secret required to activate the camera must be requested from Insta360:

```swift
guard let serialNumber = INSCameraManager.shared().currentCamera?.serialNumber else {
    self?.showAlert("提示", "请先连接相机")
    return
}
let commandManager = INSCameraManager.shared().commandManager

// You need to apply and enter your real app ID and secret.
INSCameraActivateManager.setAppid("Appid", secret: "secret")

INSCameraActivateManager.share().activateCamera(withSerial: serialNumber, commandManager: commandManager) { deviceInfo, error in
    if let activateError = error {
        self?.showAlert("Activate to \(serialNumber)", String(describing: activateError))
    } else {
        let deviceType = deviceInfo?["deviceType"] ?? ""
        let serialNumber = deviceInfo?["serial"] ?? ""
        self?.showAlert("Activate to success", "deviceType: \(deviceType), serial: \(serialNumber)")
    }
}
```

### Turn off Notification Sounds

```swift
let optionTypes = [
          NSNumber(value: INSCameraOptionsType.mute.rawValue),
          ];

let options = INSCameraOptions()
options.mute = true

INSCameraManager.socket().commandManager.setOptions(options, forTypes: optionTypes, completion: {error,successTypes in
    if let error = error {
        print("Error:\(error)")
        return
    }
    
    print("Success")
})
```

### Formatting the SD Card

```swift
let optionTypes = [
          NSNumber(value: INSCameraOptionsType.mute.rawValue),
          ];

let options = INSCameraOptions()
options.mute = true

INSCameraManager.socket().commandManager.setOptions(options, forTypes: optionTypes, completion: {error,successTypes in
    if let error = error {
        print("Error:\(error)")
        return
    }
    
    print("Success")
})
```

### Camera Lock Screen

> Note: Only supports X4 and X5

Screen Lock:

```
INSSetAccessCameraFileStateAccessStateLiveView
```

Turn off Screen Lock:

```
INSSetAccessCameraFileStateAccessStateIdle = 1
typedef NS_ENUM(NSUInteger, INSSetAccessCameraFileStateAccessState) {
  INSSetAccessCameraFileStateAccessStateUnknown = 0,

  /**没在在访问文件 */
  INSSetAccessCameraFileStateAccessStateIdle = 1,
  INSSetAccessCameraFileStateAccessStateExport = 2,
  INSSetAccessCameraFileStateAccessStateDownload = 3,
  INSSetAccessCameraFileStateAccessStatePlayback = 4,
  INSSetAccessCameraFileStateAccessStateLiveView = 5,
};

INSCameraManager.shared().commandManager.setAppAccessFileState(state, completion: { _ in
    // The callback completion can be handled here
})
```

### Turn off Camera

Command to control camera shutdown. Sample code is as follows:

Note: Only Insta360 X5 and later cameras support this feature.

```Objective-C
INSCameraManager.socket().commandManager.closeCamera({_ in
})
```

### Firmware Upgrade

Proactively upgrade by calling `updateFirwareWithOptions` interface. `INSCameraWriteFileOptions`

```markdown
@interface INSCameraWriteFileOptions : NSObject

/// file type of data to write
@property (nonatomic) INSCameraFileType fileType;

/// data of a file. if fileType is photo, data should be encoded in JPG format.
@property (nonatomic) NSData *data;

/// destinate path of the file in camera, if nil, camera will decide the uri with fileType.
@property (nonatomic, nullable) NSString *uri;

@end


- (void)updateFirwareWithOptions:(INSCameraWriteFileOptions *)options
                         completion:(void (^)(NSError * _Nullable))completion;
```

## Error Code

```objective-c
typedef NS_ENUM(NSUInteger, INSCameraErrorCode) {
    /// ok
    INSCameraErrorCodeOK = 200,
    
    /// accepted
    INSCameraErrorCodeAccept = 202,
    
    /// mainly means redirection
    INSCameraErrorCodeMovedTemporarily = 302,
    
    /// bad request, check your params
    INSCameraErrorCodeBadRequest = 400,
    
    /// the command has timed out
    INSCameraErrorCodeTimeout = 408,
    
    /// the requests are sent too often
    INSCameraErrorCodeTooManyRequests = 429,
    
    /// request is interrupted and no response has been gotten
    INSCameraErrorCodeNoResopnse = 444,
    
    INSCameraErrorCodeShakeHandeError = 445,
    
    INSCameraErrorCodePairError = 446,
    
    /// error on camera
    INSCameraErrorCodeInternalServerError = 500,
    
    /// the command is not implemented for this camera or firmware
    INSCameraErrorCodeNotImplemented = 501,
    
    /// there is no connection
    INSCameraErrorCodeNoConnection = 503,
    
    /// firmware error
    INSCameraErrorCodeFirmwareError = 504,
    
    /// Invalid Request
    INSCameraErrorCodeInvalidRequest = 505,
    
    /// bluetooth not inited
    INSCameraErrorCodeCentralManagerNotInited = 601,
};
```

# Media SDK Function

## Export Video

Video Export: `INSExportSimplify` Located in `INSCoreMedia` This class primarily simplifies video export operations, providing a set of configurable properties and methods to enable developers to customize the export process according to their needs. (Supports video export only).

### Use Instruction

This method ensures basic export functionality, with all parameters set to their default values. Users can configure relevant parameters according to specific requirements.

#### Initialization Methods

```Objective-C
-(nonnull instancetype)initWithURLs:(nonnull NSArray<NSURL>)urls outputUrl:(nonnull NSURL*)outputUrl;
```

- Function: Initializes an `INSExportSimplify` object to specify an array of input video file URLs and the output video file URL.
- Parameters:
  - `urls`: An array of input video file URLs. Cannot be empty.
  - `outputURL`: The URL of the output video file. Cannot be empty.
- Return Value: Returns an `INSExportSimplify` object.
- Sample Code:

```Objective-C
NSArray<NSURL*> *inputUrls = @[[NSURL fileURLWithPath:@"path/to/input1.mp4"], [NSURL fileURLWithPath:@"path/to/input2.mp4"]];
NSURL *outputUrl = [NSURL fileURLWithPath:@"path/to/output.mp4"];
INSExportSimplify *exporter = [[INSExportSimplify alloc] initWithURLs:inputUrls outputUrl:outputUrl];
```

#### Start Exporting

```Objective-C
-(nullable NSError*)start;
```

#### Cancel Exporting

```objective-c
-(void)cancel;
```

#### Shutdown Exporting

Cancel export. After exporting is complete, you must explicitly call the function; otherwise, it may cause a crash.

```objective-c
-(void)shutDown;
```

### Parameter Description

#### Export Resolution

This interface is used to output image resolution (aspect ratio must be 2:1).

```objective-c
/**
 * Default width and height: 1920 × 960 (The exported aspect ratio must be 2:1)
 */
@property (nonatomic) int width;
@property (nonatomic) int height;
```

#### Stitching Type

There are a total of three stitching type, namely Dynamic stitching, Disflow stitching, and AI Flow stitching.

- Dynamic Stitching: Suitable for scenes containing close-ups, or situations involving motion and rapid changes.
- Disflow Stitching: The usage scenario is identical to Dynamic stitching.
- AI Flow stitching: Based on the optimized algorithm of Insta360's existing Disflow stitching technology, this mode delivers superior stitching results. It requires configuring the stitching model path `aiStitchPath`. The model file is located in the project directory: `model_export_v22_d7c49187.mlmodelc`.

```Objective-C
@property (nonatomic) NSString *aiStitchPath;
typedef NS_ENUM(NSInteger, INSOpticalFlowType) {
    // Dynamic Stitching
    INSOpticalFlowTypeDynamicStitch = 0,
    // Disflow Stitching
    INSOpticalFlowTypeDisflow = 1,
    // AI Flow Stitching
    INSOpticalFlowTypeAiFlow = 2,
};
```

#### Stabilization Mode

The stabilization mode supports the following options in total. For panoramic cameras, there are two modes: Still and Full Directional:

- Still: No matter how you rotate the camera, the view remains unchanged.
- Full Directional: The perspective always shifts with the camera's movement.

```objective-c
/**
 * Stabilization mode, default: INSStabilizerStabModeZDirectional
 */
@property (nonatomic)INSStabilizerStabMode stabMode;
typedef NS_ENUM(NSInteger, INSStabilizerStabMode) {
    /// Turn off Stabilization
    INSStabilizerStabModeOff = -1,
    /// Still
    INSStabilizerStabModeStill,
    /// Disable yaw stabilization for directional lock; the view can only move around the z-axis (pan movement).
    INSStabilizerStabModeZDirectional,
    /// Enable image stabilization and enable horizontal correction, i.e., full-directional
    INSStabilizerStabModeFullDirectional,
    ///Without calibrating the horizon, i.e., free-footage, total free camera mode, camera can move through three axis
    INSStabilizerStabModeFreeFootage,
    ///  //Process flipping
    INSStabilizerStabModeFlipEffect,
    ///Enable image stabilization and average attitude correction, i.e.,relative-refine
    INSStabilizerStabModeRelativeRefine,
    ///Enable anti-shake, enable level correction, i.e., absolute-refine
    INSStabilizerStabModeAbsoluteRefine,
    /// Bullet Time
    INSStabilizerStabModeBulletTime,
    /// Pano Fpv
    INSStabilizerStabModePanoFPV
};
```

#### Lens Guards Type

If wearing lens guards during filming, configure this parameter according to the type. (Standard lens guards in the store is rated Class A, while premium lens guards is rated Class S.)

```objective-c
/**
 * Lens Guards
 */
@property (nonatomic) INSOffsetConvertOptions protectType;
typedef NS_OPTIONS(NSUInteger, INSOffsetConvertOptions) {
    INSOffsetConvertOptionNone                   = 0,
    INSOffsetConvertOptionEnableWaterProof       = 1 << 0, // ONE X lens guards
    INSOffsetConvertOptionEnableDivingAir        = 1 << 1, // Invisible dive case (Air)
    INSOffsetConvertOptionEnableDivingWater      = 1 << 2, // Invisible dive case（Water）
    INSOffsetConvertOptionEnableDivingAirV2      = 1 << 3, // Invisible dive case (Air, V2)
    INSOffsetConvertOptionEnableDivingWaterV2    = 1 << 4, // Invisible dive case (Water, V2)
    INSOffsetConvertOptionEnableBuckleShell      = 1 << 5, // Buckle shell
    INSOffsetConvertOptionEnableAdhesiveShell    = 1 << 6, // Adhesive shell
    INSOffsetConvertOptionEnableGlassShell       = 1 << 7, // Premium lens guards (S)
    INSOffsetConvertOptionEnablePlasticCement    = 1 << 8, // Standard lens guards (A)
    INSOffsetConvertOptionEnableAverageShell     = 1 << 9  // A/S average shell
};
```

#### Color Fusion

```objective-c
/**
 * Color Fusion: Default false
 */
@property (nonatomic) BOOL colorFusion;
```

The primary causes of chromatic aberration are as follows:

- First, since the two lenses operate independently, their respective video exposures may not align perfectly. When stitched together, noticeable brightness differences can occur. This stems from variations in capturing parameters and sensor performance between lenses, resulting in differing light intensities captured.
- Second, uneven lighting on either side of the lens leads to differing camera exposures, sometimes resulting in noticeable brightness differences between the front and rear lens footage. This phenomenon is particularly pronounced in high-contrast lighting scenarios, such as areas where bright light meets shadow, or environments with significant indoor-outdoor lighting disparities. In these cases, the camera struggles to balance light intensity across different zones, causing substantial brightness variations in the captured images.

To address these issues, Color Fusion technology was developed. Through a series of algorithms and image processing techniques, it adjusts and optimizes footage captured by different lenses to reduce or eliminate brightness discrepancies. This ensures the stitched video appears more natural and seamless.

#### Defringe

This interface features a function to eliminate specific phenomena, specifically designed to counteract purple fringing caused by lighting conditions during recording. In real-world recording scenarios where lighting is complex and variable, this interface effectively addresses purple fringing issues arising from two common lighting situations. The first scenario involves intense outdoor sunlight. When recording during bright daylight hours, high-intensity outdoor light easily triggers purple fringing. This interface effectively eliminates fringing in such conditions. The second scenario pertains to indoor lighting. Light emitted by various indoor sources, such as incandescent bulbs or fluorescent lights, can also cause purple fringing in recorded footage. This interface similarly eliminates fringing in these indoor lighting scenarios. 

Usage: To utilize this algorithm, configure the corresponding model managed via a JSON file: `coreml_model_v2_scale6_scale4.json`. Copy these models and JSON files into the app. Refer to the following code for parameter configuration:

Video

```javascript
let jsonFilePath: String = Bundle.main.path(forResource: "coreml_model_v2_scale6_scale4", ofType: "json")!
        let components = jsonFilePath.split(separator: "/")
        let modelFilePath = components.dropLast().joined(separator: "/")
        
        var jsonContent = ""
        do {
            let data = try Data(contentsOf: URL(fileURLWithPath: jsonFilePath))
            jsonContent = String(data: data, encoding: .utf8) ?? ""
        } catch {
            assertionFailure("\(jsonFilePath)文件无法读取")
        }
     
        let initInfo = INSDePurpleFringeFilterInitInfo.init(imageTransferType: .AUTO, algoAccelType: .AUTO)
        
        let depurpleFringInfo = INSDePurpleFringeFilterInfo()
        depurpleFringInfo.setInit(initInfo)
        depurpleFringInfo.setModelJsonContent(jsonContent)
        depurpleFringInfo.setModelFilePath(modelFilePath)
```

Photo

```javascript
       let jsonFilePath: String = Bundle.main.path(forResource: "coreml_model_v2_scale6_scale4", ofType: "json")!
            let components = jsonFilePath.split(separator: "/")
            let modelFilePath = components.dropLast().joined(separator: "/")
            var jsonContent = ""
            do {
                let data = try Data(contentsOf: URL(fileURLWithPath: jsonFilePath))
                jsonContent = String(data: data, encoding: .utf8) ?? ""
            } catch {
                assertionFailure("\(jsonFilePath)文件无法读取")
            }
            
            let depurpleFringInfo = INSExporter2DefringConfiguration.init()
            depurpleFringInfo.modelJsonContent = jsonContent
            depurpleFringInfo.modelFilePath = modelFilePath
            depurpleFringInfo.enableDetect = true
            depurpleFringInfo.useFastInferSpeed = false
            videoExporter?.defringeConfig = depurpleFringInfo
```

#### Denoise

In the field of video processing, multi-frame denoising technology is a crucial image processing method that can reduce or eliminate noise in video frames. Noise adversely affects image quality and viewing experience. Unlike single-frame denoising, which processes only a single frame, multi-frame denoising leverages redundant information from preceding and subsequent frames to more accurately distinguish between image content and noise. This approach preserves image details while removing noise, resulting in high-quality footage. However, multi-frame denoising also has drawbacks. It requires processing multiple frames, involving complex computational analysis that consumes system resources. This may cause devices to slow down or experience lag, while also reducing video export speed and increasing time costs.

```Objective-C
/**
 * Denoise
 * This property controls whether to enable multi-frame noise reduction for video. When set to YES, noise reduction is enabled; when set to NO, noise reduction is disabled.
 */
@property (nonatomic) BOOL enableDenoise;
```

#### Color Enhancement

```Objective-C
/**
 * Color enhancement, disabled by default
 */
@property (nonatomic) BOOL clolorPlus;
```

#### Progress and Status Callbacks

After exporting begins, the `INSRExporter2ManagerDelegate` protocol will provide real-time callbacks regarding the export progress and status.

```Objective-C
typedef NS_ENUM(NSInteger, INSExporter2State) {
    INSExporter2StateError = -1, // Indicates that an error occurred during the export process.
    INSExporter2StateComplete = 0, // Indicates that the export has been successfully completed.
    INSExporter2StateCancel = 1, // Indicates that the user manually canceled the export operation.
    INSExporter2StateInterrupt = 2, // Indicates that the export was unexpectedly interrupted.
    INSExporter2StateDisconnect = 3, // Indicates disconnection from the server
    INSExporter2StateInitError = 4, // Indicates an error occurred during initialization.
};
@protocol INSRExporter2ManagerDelegate  <NSObject>

- (void)exporter2Manager:(INSExporter2Manager *)manager progress:(float)progress;

- (void)exporter2Manager:(INSExporter2Manager *)manager state:(INSExporter2State)state error:(nullable NSError*)error;

// Used for panoramic tracking detection (generally not required)
- (void)exporter2Manager:(INSExporter2Manager *)manager correctOffset:(NSString*)correctOffset errorNum:(int)errorNum totalNum:(int)totalNum clipIndex:(int)clipIndex type:(NSString*)type;
@end
```

### Error Code

```Objective-C
typedef NS_ENUM(NSInteger, INSExporter2State) {
    INSExporter2StateError = -1, // Indicates that an error occurred during the export process.
    INSExporter2StateComplete = 0, // Indicates that the export has been successfully completed.
    INSExporter2StateCancel = 1, // Indicates that the user manually canceled the export operation.
    INSExporter2StateInterrupt = 2, // Indicates that the export was unexpectedly interrupted.
    INSExporter2StateDisconnect = 3, // Indicates disconnection from the server
    INSExporter2StateInitError = 4, // Indicates an error occurred during initialization.
};
};
```

## Photo Export

### Instruction for Use

Photo Export: `The INSExportImageSimplify` class belongs to the `INSCoreMedia` framework. This class aims to simplify photo export operations by providing a set of configurable properties and methods. It supports exporting both `standard photos` and `HDR photos`, allowing developers to customize the export process according to their requirements.

#### Standard Photos Export

```Objective-C
- (NSError*)exportImageWithInputUrl:(nonnull NSURL *)inputUrl outputUrl:(nonnull NSURL *)outputUrl;
```

- **Functions**: Exports standard photos, supporting both remote and local.
- **Parameters**:
  - `inputURL`: URL of the input image. Cannot be empty.
  - `outputURL`: URL of the output image. Cannot be empty.
- **Return Value**: Returns an NSError object if an error occurs during export; returns nil if export succeeds.
- **Sample Code**:

```Objective-C
INSExportImageSimplify *exporter = [[INSExportImageSimplify alloc] init];
NSURL *inputUrl = [NSURL fileURLWithPath:@"path/to/input.jpg"];
NSURL *outputUrl = [NSURL fileURLWithPath:@"path/to/output.jpg"];
NSError *error = [exporter exportImageWithInputUrl:inputUrl outputUrl:outputUrl];
if (error) {
    NSLog(@"导出失败：%@", error.localizedDescription);
} else {
    NSLog(@"导出成功");
}
```

#### HDR image compositing method (supports only local HDR images)

> For X3 and earlier models, HDR needs to be composited externally. After shooting, please download the HDR footage to your local device and then use this interface to composite it.

```Objective-C
- (NSError*)exportHdrImageWithInputUrl:(nonnull NSArray<NSURL*> *)inputUrl outputUrl:(nonnull NSURL *)outputUrl;
```

- Function:
  - This function is primarily used for exporting high dynamic range (HDR) images. It supports only local photos, which must be stored on the local device to perform the export operation. Additionally, to ensure the quality and effectiveness of the export, a minimum of 3 input photos is required.
- Parameters:
  - `inputURL`: An array of URLs for input photos. This array cannot be empty, and its elements must be greater than or equal to 3. These URLs point to local photos requiring HDR export.
  - `outputURL`: Specifies the URL for the output photo. Similarly, this parameter cannot be empty and determines the storage location for the exported HDR photo.
- Return Values:
  - During export, any errors encountered will return an `NSError` object. This object contains detailed error information for developer error handling and debugging.
  - If export succeeds without errors, the return value will be `nil`, indicating the operation completed successfully.
- Sample Code:

```Objective-C
INSExportImageSimplify *exporter = [[INSExportImageSimplify alloc] init];
NSArray<NSURL*> *inputUrls = @[[NSURL fileURLWithPath:@"path/to/input1.jpg"], [NSURL fileURLWithPath:@"path/to/input2.jpg"], [NSURL fileURLWithPath:@"path/to/input3.jpg"]];
NSURL *outputUrl = [NSURL fileURLWithPath:@"path/to/output_hdr.jpg"];
NSError *error = [exporter exportHdrImageWithInputUrl:inputUrls outputUrl:outputUrl];
if (error) {
    NSLog(@"导出失败：%@", error.localizedDescription);
} else {
    NSLog(@"导出成功");
}
```

### Parameters Description

参考视频参数，图片参数是视频参数的子集。区别点：图片导出没有回调方法，直接通过返回值返回错误信息。

Refer to the video parameters; image parameters are a subset of video parameters. Key difference: Image export does not use callback methods; error messages are returned directly via the return value.

### Error Code

```Plain
typedef NS_ENUM(NSInteger, INSExportSimplifyError) {
    INSExportSimplifyErrorSuccess = 0,// Success
    INSExportSimplifyErrorInitFailed = 3001, // Export failed
    INSExportSimplifyErrorImageInitFailed = 4001, // Initialization failed
};
```

## Logging

Within the `INSCoreMedia` library, log management tasks can be accomplished using the `INSRegisterLogCallback` class. This class offers a rich set of functionalities. First, it supports flexible configuration of log output paths. Developers can direct logs to specified files or storage locations based on project requirements, facilitating centralized log review and analysis later. Second, it features log level filtering. Logs of different levels—such as debug, info, warning, and error—hold varying importance during different stages of project development and operation. By configuring filter levels, developers can output only logs of specific levels or higher, reducing unnecessary log clutter and enhancing development and debugging efficiency.

Additionally, `INSCoreMedia` supports callback registration. Users can register callback functions to implement custom log processing within these callbacks. For instance, logs can be printed to the console in callbacks, enabling developers to view log content in real-time during debugging. Alternatively, logs can be stored as log files for detailed retrospective analysis and tracing, providing robust data support for project maintenance and optimization.

Configuring log output files:

```Objective-C
/**
     1. If the configuration includes file output for logging, the file path must be set.
     2. Set the output log level; logs of this level or higher will be output.
 */
- (void)configLogKind:(InsLogKind)kind minLogLevel:(InsLogLevel)level optionalFullFilePath:(NSString * __nullable)fullFilePath;
```

Registration callback: 

```objective-c
/**
  Custom log received callbacks
*/
- (void)registerLogCallBack:(void(^)(NSString *_Nonnull tag, InsLogLevel level, NSString * _Nonnull fileName, NSString * _Nonnull funcName, int line, NSString *_Nonnull message))block;
- (void)registerLogCallBackBmg:(void(^)(InsLogLevel level,NSString *_Nonnull message))block;
```

# FAQ

## Issues Related to URI Returned by HDR Capturing

1. The x4 and x5 models support in-camera HDR merging. When in-camera merging is enabled, HDR synthesis occurs within the camera, resulting in a single merged image after processing.
2. Cameras up to and including the X3 return multiple URIs when capturing HDR images. However, newer cameras such as the X4 and X5 return only one URI regardless of whether in-camera merging is enabled (not supported at the firmware level).