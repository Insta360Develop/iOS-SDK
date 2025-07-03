# Note:
Currently on `V1.8.2-build4` branch. To view other versions, please switch branches.

# How to get?

Please visit [https://www.insta360.com/sdk/apply](https://www.insta360.com/sdk/apply) to apply for the latest SDK.

# Support

Developers' Page: https://www.insta360.com/developer/home  
Insta360 Enterprise: https://www.insta360.com/enterprise  
Issue Report: https://insta.jinshuju.com/f/hZ4aMW  

# [中文文档](README_zh.md)

# Overview

The iOS SDK is mainly used to connect, set and obtain camera parameters, control the camera to take pictures and record, download files, upgrade firmware, and support video and image export, etc.

Supported models: X5, X4, X3, ONE X2, ONE X, ONE RS, ONE RS 1-Inch.

# Table of contents
* [INSCameraSDK](#inscamerasdk)
	* [Environmental preparation](#environmental-preparation)
	* [Camera connection and status](#camera-connection-and-status)
	* [Shooting mode](#shooting-mode)
	* [Camera parameters](#camera-parameters)
	* [Photo Capture](#photo-capture)
	* [Video Recording](#video-recording)
	* [Preview](#preview)
	* [Other features](#other-features)
* [INSMediaSDK](#insmediasdk)
	* [Export (video)](#export-video)
	* [Export (image)](#export-image)

# INSCameraSDK

## Environmental preparation

### Connect the camera via Wi-Fi

- **Enable camera Wi-Fi function**

  Enable Wi-Fi function on the camera.

- **Connect to Wi-Fi**

  In the Wi-Fi settings of your phone, find and select the corresponding camera name.

- **Enter connection password**

| Model    | Password acquisition method                                  |
| -------- | ------------------------------------------------------------ |
| X2       | Fixed password：`88888888`                                   |
| X3/X4/X5 | Manual acquisition: Camera screen → Settings → Wi-Fi → Show password<br>Bluetooth acquisition: see 1.2.3 |

### Connect the camera via Bluetooth

After connecting the camera via Bluetooth, you can use shooting, recording and other functions normally, but it does not support real-time preview and file download. If you need preview function, you can get the WIFI name and password by calling the corresponding interface after Bluetooth pairing is successful, and then connect via wifi.

#### Search for Bluetooth devices

Call the following method to scan Bluetooth devices

```python
INSBluetoothManager().scanCameras
```

#### Establish a Bluetooth connection

Establish a Bluetooth connection using the following interface:

```c++
- (id)connectDevice:(INSBluetoothDevice *)device
         completion:(void (^)(NSError * _Nullable))completion;
```

#### Get camera wifi account and password

After successful Bluetooth connection, call the interface to obtain Wi-Fi information. Example code is as follows:
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

## Camera connection and camera status

### Connect the camera

After connecting the camera via Wi-Fi, the interface in INSCameraSDK can be called to establish the connection. The interface uses singleton mode, and an instance is created when the first call is made, and the same instance is used to reuse in subsequent calls.

```c++
INSCameraManager.socket().setup()
```

### Disconnect the camera

When you need to disconnect from the camera, please call the following interface:

```c++
INSCameraManager.socket().shutdown()
```

### Checking camera connection status

Obtain the connection status of the camera through the following methods:

```swift
INSCameraManager.socket().cameraState
```

Returning to the status, if the status is `INSCameraStateConnected`indicates that the connection has been successfully established. The status is defined as follows:

```c++
INSCameraManager.socket().cameraState

// There are several states in total
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

## Shooting mode

This section primarily introduces how to configure and retrieve various camera parameters, including shooting modes and photographic settings. The parameters are generally categorized into two types: numerical parameters (such as ISO, EV value, and shutter speed) and enumerated parameters (such as shooting mode, exposure mode, and resolution).

### Get the current shooting mode

Use the interface `INSCameraManager.shared ().commandManager.getOptionsWithTypes`to obtain the shooting mode of the current camera. This method requires passing the following parameters:

* **optionTypes**
  Specifies which parameters need to be retrieved, with each parameter corresponding to an enumeration value (see `INSCameraOptionsType`) that needs to be converted to an `NSNumber` array.
* **requestOptions**
  Maximum configurable request timeout (default 10 seconds).
* **completion**
  Return in callback:
  * `error`: `nil` indicates successful acquisition, otherwise contains the reason for the error.
  * `cameraOptions`: Contains the current camera shooting mode (shooting mode is `photoSubMode` or `videoSubMode`, see below for specific enumeration meanings).

Enumeration of shooting modes:
```typescript
// Take photo
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
// Record video
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

### Set shooting mode

Configure the specified mode to the camera through the method: `INSCameraManager.shared ().commandManager.setOptions.` There are several important points in setting the shooting mode

* **INSCameraOptions**
  It is used to carry the shooting mode configuration, mainly including the following two attributes:
  * `photoSubMode`：Camera mode
  * `videoSubMode`：Video mode

* **INSCameraOptionsType**
  Used to specify which parameters in `INSCameraOptions` need to take effect.

  * Camera mode：`INSCameraOptionsType::INSCameraOptionsTypePhotoSubMode`
  * Video mode：`INSCameraOptionsType::INSCameraOptionsTypeVideoSubMode`

The following is an example code (using setting the image shooting mode as an example):

```c++
// Picture shooting mode enumeration
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

## Camera parameters

Camera parameters are usually divided into two categories. The first is a specific numerical value, such as exposure ISO, EV value, shutter speed, etc. The second is an enumeration value, such as shooting mode, exposure mode, etc. For the former, you can directly assign values to the parameters in the `INSPhotographyOptions` and send them to the camera. For the latter, you need to check the meaning represented by the corresponding enumeration in the table, and then assign the corresponding enumeration value to the `INSPhotographyOptions`. The following is the relationship between resolution and its corresponding enumeration value. Other parameters can be viewed through the `"common_camera_setting_proto json"` file in the SDK project.

```swift

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

### Get camera parameters

Get the camera through the method: `getPhotographyOptions` get. The main imported parameters required are three `functionMode` and `optionTypes`, as well as a closure callback.
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

* `functionMode`: Indicates the current shooting mode
* `optionTypes`: Each parameter corresponds to an enumeration value.
  * Corresponding to resolution: INSPhotographyOptionsTypePhotoSize = 40
  * White balance corresponds to: INSPhotographyOptionsTypeWhiteBalance = 13
* Closure returns an `NSError` and `INSPhotographyOptions`
  * `NSError` is `nil` indicating success, otherwise it will include the reason for failure
  * `INSPhotographyOptions` contains the specific values of the current camera parameters, such as resolution, white balance, etc

Sample code is as follows: Get the value of the current camera white balance
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


### Set camera parameters
To set parameters for the camera, you need to use the following method:`INSCameraManager.shared().commandManager.setPhotographyOptions`, the main imported parameters are as follows.

* `INSPhotographyOptions`: Store configured parameters

* `INSPhotographyOptionsType`: Indicates which parameters need to be configured (each parameter corresponds to an enumeration).

* `INSCameraFunctionMode`: Current shooting mode, current shooting mode.
  * Camera settings required shooting mode `INSCameraFunctionMode` and get from the camera shooting mode `INSPhotoSubMode` or `INSVideoSubMode` can not be directly converted, the corresponding relationship reference project file class: `HUMCaptureMode`. `HUMCaptureMode` will be the camera current mode INSPhotoSubMode or INSVideoSubMode into a unified INSCameraFunctionMode.
  
* Closure callback: return whether the acquisition was successful

> All parameter types can only be set to parameters supported on the camera screen and cannot be set arbitrarily

Example code is as follows: Set specified white balance

```swift
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

### Numeric type

#### White balance

* The properties in `INSPhotographyOptions` are:
```objective-c
@property (nonatomic) uint32_t whiteBalanceValue;
```
* The corresponding enumeration in `INSPhotographyOptionsType` is:
```objective-c
INSPhotographyOptionsTypeWhiteBalance = 13
```
* Suppose the current shooting mode `INSCameraFunctionMode 6 `represents normal photography.

#### ExposureBias (Only auto exposure supports "AUTO")

> Adjustment supported only in exposure mode "AUTO" "FULL\_AUTO"

* `INSPhotographyOptions` properties are: `exposureBias`
```objective-c
@property (nonatomic) float exposureBias;
```

* The corresponding enumeration in `INSPhotographyOptionsType` is:
```objective-c
INSPhotographyOptionsTypeExposureBias = 7
```

#### ISO，shutterSpeed

> Adjustment is supported only in exposure modes "MANUAL" and "ISO\_PRIORITY"

* `INSPhotographyOptions` properties are: `stillExposure.iso`, `stillExposure.shutterSpeed`

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

* The corresponding enumeration in `INSPhotographyOptionsType` is:

```objective-c
INSPhotographyOptionsTypeStillExposureOptions = 20
```

#### lapse_time

This parameter is only effective in timelapse mode.

```objective-c
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

### Enumeration and Bool

#### Resolution

Since 1.8.1, the resolution of the version is uniformly managed by enumeration. You need to manually configure `isUseJsonOptions` to `true`

```Objective-C
INSPhotographyOptions.isUseJsonOptions = true
```

##### Photo resolution

* `INSPhotographyOptions` properties are: photoSizeForJson
* `PhotoSizeForJson = 17`, indicating that the resolution of the X4 camera is `8192 x 6144`.
* The corresponding enumeration in `INSPhotographyOptionsType` is:

```objective-c
INSPhotographyOptionsTypePhotoSize = 40
```

* Suppose the current shooting mode `INSCameraFunctionMode 6` represents normal photography.

Code example:
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

Correspondence between resolution and enumeration values
```swift
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

##### Record

- The property in `INSPhotographyOptions` is: `photoSizeForJson`
- The mapping table is too long and is not shown here. Please move to the `record_resolution` list in the `common_camera_setting_proto.json` file

#### Exposure mode

* The properties in `INSPhotographyOptions` are:
```objective-c
@property (nullable, nonatomic) INSCameraExposureOptions *stillExposure;


@interface INSCameraExposureOptions : NSObject

@property (nonatomic) uint8_t program; // Exposure mode

@property (nonatomic) NSUInteger iso;

#if TARGET_OS_IOS
@property (nonatomic) CMTime shutterSpeed;
#endif

@end
```

* The corresponding enumeration in `INSPhotographyOptionsType` is:
```objective-c
INSPhotographyOptionsTypeStillExposureOptions = 20
```

```json
"exposure_program": {
   "AUTO": "0",
   "ISO_PRIORITY": "1",
   "SHUTTER_PRIORITY": "2",
   "MANUAL": "3",
   "ADAPTIVE": "4",
   "FULL_AUTO": "5"
},
```

> Support setting multiple parameters to the camera at the same time, as follows：
```swift
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
#### Pureshot

> Note: X5 has PureShot turned on by default and does not support turning it off

* The properties in `INSPhotographyOptions` are:

```objective-c
@property (nonatomic) uint8_t rawCaptureType;
```

* The corresponding enumeration in `INSPhotographyOptionsType` is:

```objective-c
INSPhotographyOptionsTypeRawCaptureType = 25,
```

* Enumeration type:

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

#### In-camera Stitching (Bool)

When taking pictures, stitch panoramic images inside the camera without the need for external stitching. Only cameras after X4 are supported.

```swift
var typeArray = [INSCameraOptionsType]()
let cameraOptions = INSCameraOptions()

// Open is true, close is false
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

## Photo Capture

Send shooting instructions to the camera through the `"INSCameraManager.shared ().commandManager.takePicture"` method. If the error is nil, it means the shooting is successful, otherwise it contains error information. The remote access address of the photo is included in optionInfo, which can be exported remotely (without downloading to the local).&#x20;
```bash
INSCameraManager.shared().commandManager.takePicture(with: nil, completion: { (error, optionInfo) in
    
    print("end takePicture \(Date())")     
    guard let uri = optionInfo?.uri else {
        return
    }
    print("Take Picture Url:\(uri)")\
})
```

## Video Recording

Start recording:
```python
INSCameraManager.shared().commandManager.startCapture(with: nil, completion: { (err) in
    if let err = err {
        self.showAlert(row.tag!, "\(String(describing: err))")
        return
    }
})
```

Stop recording:
```python
INSCameraManager.shared().commandManager.stopCapture(with: nil, completion: { (err, info) in
    if let err = err {
        self.showAlert(row.tag!, "\(String(describing: err))")
        return
    }
})
```

## Preview

Preview must be done after establishing a connection with the camera. There are several core classes. Please refer to the demo for details: `CameraConfigByJsonController`.

* `INSCameraSessionPlayer`
  * Bind bounds on the iOS side.
  * Implementation Agreement: `INSCameraSessionPlayerDelegate`.
  * Implementation protocol: `INSCameraSessionPlayerDataSource`, it is recommended to implement the following two protocols:
	  Configure offset information: `updateOffsetToPlayer`.
	  Configure stabilization information: `updateStabilizerParamToPlayer`.
* `INSCameraMediaSession`
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

### Preview real-time anti-shake data acquisition

During the preview process, by implementing the `INSCameraSessionGyroDelegate` protocol in `INSCameraSessionPlayer`, real-time callback of stabilization data can be achieved. The callback data has two formats, one is the original data `NSData`, and the other is the parsed `INSGyroRawItem` format.

```objective-c
@protocol INSCameraSessionGyroDelegate <NSObject>

@optional

- (void)onRawGyroData:(NSData *)rawData timestampMs:(int64_t)timestampMs;

- (void)onParsedGyroData:(NSMutableArray<INSGyroRawItem *> *)gyroItems timestampMs:(int64_t)timestampMs;

@end
```

```Objective-C
@interface INSCameraSessionPlayer : NSObject<INSCameraPlayerSession>

@property (nonatomic, weak  ) id<INSCameraSessionGyroDelegate> gyroDelegate;

@end
```

## Other features

### Firmware upgrade

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

### Activate

`Appid` and `secret` need to be requested from Insta360.

```swift
guard let serialNumber = INSCameraManager.shared().currentCamera?.serialNumber else {
    self?.showAlert("提示", "请先连接相机")
    return
}
let commandManager = INSCameraManager.shared().commandManager

// "提示" means "Note", "请先连接相机" means "Please connect the camera first."

//You need to apply, fill in your real appid and secret.
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

### Write GPS data

#### Picture

When calling the shooting interface, pass the `INSMediaGps` data into the shooting parameters:

```JavaScript
func pictureGps() {
    // Construct the current location (GPS) data. Here we use the Hong Kong coordinate point as an example.
    let newestLocation = CLLocation(
        coordinate: CLLocationCoordinate2D(latitude: 22.2803, longitude: 114.1655), // Latitude and longitude
        altitude: 10.0,              // Altitude (m)
        horizontalAccuracy: 5.0,     // Horizontal accuracy (meters)
        verticalAccuracy: 5.0,       // Vertical accuracy (meters)
        timestamp: Date()            // current timestamp
    )

    // Create extended metadata objects (for attaching GPS, gyroscope, magnetometer, etc. information)
    let metaData = INSExtraMetadata()

    // Create a media GPS information object using the constructed CLLocation
    let mediaGps = INSMediaGps(clLocation: newestLocation, isValidLocation: true)

    // Write GPS information into metadata
    metaData.gps = mediaGps

    // Create an extended information object and set metadata (gyroscope data is not set here)
    let extraInfo = INSExtraInfo(
        version: Int32(INSExtraInfoVersion.one2.rawValue), // Use extended information version one2
        metadata: metaData,                                 // Metadata with GPS information
        gyroData: nil                                       // No gyroscope data
    )

    // Construct a camera options object, including extended information
    let options: INSTakePictureOptions = INSTakePictureOptions(extraInfo: extraInfo)

    // Call the camera to take a photo
    INSCameraManager.shared().commandManager.takePicture(with: options, completion: { (error, optionInfo) in
        print("end takePicture \(Date())") // Print photo completion time

        // Extract the image URI from the response
        guard let uri = optionInfo?.uri else {
            return // If the acquisition fails, return directly
        }

        print("Take Picture Url:\(uri)") // Print the address of the captured photo file
    })
}
```

#### Video

After starting recording, upload the constructed GPS data to the camera via `uploadGpsDatas`:



```JavaScript
// Create additional information object (generally used to pass extended parameters)
let extraInfo = INSExtraInfo()

// Create a shooting parameter object and set extraInfo
let options = INSCaptureOptions(extraInfo: extraInfo)

// Start shooting
self?.commandsManager.startCapture(with: options, completion: { (err) in
    if let err = err {
        // Startup failed, pop-up window prompts
        print("\(row.title!) failed with error: \(err)")
        self?.showAlert(row.title!, err.localizedDescription)
        return
    }

    // Perform GPS data upload after 5 seconds delay
    DispatchQueue.main.asyncAfter(deadline: .now() + 5) {

        // Create an array of GPS data (simulate 100)
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

        // Convert to binary data (optional: for debugging or uploading)
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

            // Stop recording after uploading is complete
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



### Get SD card status

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

### Get current battery level

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

### Get activation time

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

### Silent switch

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

### Serial number

It can be called after establishing a connection with the camera
```swift
INSCameraManager.shared().currentCamera?.serialNumber
```

### Firmware version number

```swift
INSCameraManager.shared().currentCamera?.firmwareRevision
```

### Turn off camera
The command to control the camera to close, currently only supports the X5 camera. The example is as follows:
```swift
INSCameraManager.socket().commandManager.closeCamera({_ in})
```

### LOG management

In `INSCameraSDK`, all the logs generated by it will be passed to the upper layer in the form of callbacks by means of the `INSCameraSDKLoggerProtocol` protocol. For the upper layer, after receiving these logs, it can choose to print out the logs, which is convenient for developers to view the relevant information in the running process of the SDK during the debugging phase, and can also store the logs for subsequent Data Analysis or troubleshooting. If you want to know more detailed operations and configurations, you can refer to the `CameraConfigByJsonController`.

INSCameraSDKLogger takes a singleton as the carrier of the `INSCameraSDKLoggerProtocol`, and the `logDelegate` property needs to be configured to carry `INSCameraSDKLoggerProtocol` callbacks.

```swift
@protocol INSCameraSDKLoggerProtocol

- (void)logError:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logWarning:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logInfo:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logDebug:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;

- (void)logCrash:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;

@end
```

### Camera notification

During the operation of the camera, it may encounter low battery, memory card abnormality, recording interruption, high temperature, etc. In order to notify the application in time for response processing, the camera sends specific notifications through `NotificationCenter`. Developers need to register corresponding notifications when the program is initialized, monitor and handle these abnormal conditions to ensure stable operation of the device and improve user experience.

#### Low Battery

- **Notification string:** `INSCameraBatteryLowNotification`

```Swift
NotificationCenter.default.addObserver(self,
                                       selector: #selector(self.batteryLowNotification(_:)),
                                       name: NSNotification.Name.INSCameraBatteryLow,
                                       object: nil)
```

**Callback example:**

```Swift
@objc func batteryLowNotification(_ notification: Notification) {
    if let status = notification.userInfo?["storage_status"] as? INSCameraBatteryStatus {
        self.showAlert("Notification", "Battery Low: \(status.batteryScale)%")
    }
    print("Notification: Battery Low")
}
```

**Note:** When receiving this notification, it is recommended that the user replace or charge the device in time to prevent the device from being unable to save data due to power outage.

#### SD card status notification

- **Notification string:** `INSCameraStorageStatusNotification`

```Objective-C
NotificationCenter.default.addObserver(self,
                                               selector: #selector(self.storageFullNotification(_:)),
                                               name: NSNotification.Name.INSCameraStorageStatus,
                                               object: nil)
```

**Status enumeration:**

```Objective-C
typedef NS_ENUM(NSInteger, INSPBCardState) {
    INSPBCardState_GPBUnrecognizedEnumeratorValue = kGPBUnrecognizedEnumeratorValue,
    INSPBCardState_StorCsPass = 0,         // 正常
    INSPBCardState_StorCsNocard = 1,        // 无卡
    INSPBCardState_StorCsNospace = 2,       // 空间不足
    INSPBCardState_StorCsInvalidFormat = 3, // 格式错误
    INSPBCardState_StorCsWpcard = 4,        // 写保护
    INSPBCardState_StorCsOtherError = 5     // 其他错误
};
```

**Callback example:**

```Swift
@objc func storageFullNotification(_ notification: Notification) {
    
    if let status = notification.userInfo?["card_state"] as? INSCameraCardState {
        self.showAlert("Notification", "Battery Low :\(status)")
    }
    
    print("Notification: Storage Full")
}
```

#### Recording exception

- **Notification string**: `INSCameraCaptureStoppedNotification`

```Objective-C
NotificationCenter.default.addObserver(self,
                                        selector: #selector(self.captureStoppedNotification(_:)),
                                        name: NSNotification.Name.INSCameraCaptureStopped,
                                        object: nil)
```

**Callback example:**

```Swift
@objc func captureStoppedNotification(_ notification: Notification) {
    
    if let info = notification.userInfo?["video"] as? INSCameraVideoInfo, let errorCode = notification.userInfo?["error_code"] as? Int{
        self.showAlert("Notification", "Capture Stop, error code:\(errorCode)")
    }
    
    print("Notification: Capture Stop")
}
```

**Note:** If an abnormality occurs during recording, such as storage abnormality or overtemperature, the device will automatically stop recording. At this time, the user should be prompted to re-operate to avoid losing important content.

#### High temperature alarm

- **Notification string:** `INSCameraTemperatureStatusNotification`

```Objective-C
NotificationCenter.default.addObserver(self,
                                        selector: #selector(self.temperatureValueNotification(_:)),
                                        name: NSNotification.Name.INSCameraTemperatureStatus,
                                        object: nil)
```

**Callback example:**

```Swift
@objc func temperatureValueNotification(_ notification: Notification) {
    
    if let status = notification.userInfo?["temperature_status"] as? INSCameraTemperatureStatus {
        self.showAlert("Notification", "Temperature warning, \(status.temperature) ")
    }
    
    print("Notification: Temperature warning")
}
```

**Note:** High temperatures may cause device performance degradation or even damage. Upon receiving this notice, users should be advised to stop using the device and wait until it cools down before continuing to use it.

### Error code

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

# INSMediaSDK

## Export (video)

Video Export: Provides a simplified export method `INSExportSimplify` specifically for tob users, located in `INSCoreMedia`. This class is mainly used to simplify the operation of video export, providing a series of configurable properties and methods to facilitate developers to customize the export process according to their needs. (Only supports exporting videos).
### Instructions for use

This method can ensure the basic export effect, and all parameters used for export are default values. Users can configure relevant parameters according to specific needs.

#### Initialization method

```objective-c
-(nonnull instancetype)initWithURLs:(nonnull NSArray<NSURL>)urls outputUrl:(nonnull NSURL*)outputUrl;
```
* **Function**: Initialize the `INSExportSimplify` object to specify the URL array of the input video file and the URL of the output video file.

* **Parameters**:
  * `urls`: Enter the URL array of the video file, it cannot be empty.
  * `outputUrl`: Output the URL of the video file, cannot be empty.

* **Return value**: Returns an `INSExportSimplify` object.

* **Example code:**

```objective-c
NSArray<NSURL*> *inputUrls = @[[NSURL fileURLWithPath:@"path/to/input1.mp4"], [NSURL fileURLWithPath:@"path/to/input2.mp4"]];
NSURL *outputUrl = [NSURL fileURLWithPath:@"path/to/output.mp4"];
INSExportSimplify *exporter = [[INSExportSimplify alloc] initWithURLs:inputUrls outputUrl:outputUrl];
```

#### Start exporting

```objective-c
-(nullable NSError*)start;
```

#### Cancel export

```objective-c
-(void)cancel;
```

#### Destroy export

Cancel the export. After the export is completed, be sure to call it actively, otherwise it may cause a crash.

```objective-c
-(void)shutDown;
```

### Parameter description

#### Export resolution

This interface is used to output image resolution (aspect ratio must be 2:1)
```objective-c
/**
 * Default width and height: 1920 * 960 (the exported image ratio needs to be 2:1).
 */
@property (nonatomic) int width;
@property (nonatomic) int height;
```

#### **Stitching Type**（opticalFlowType）

There are a total of four stitching type, namely template stitching, dynamic stitching, optical flow stitching, and AI stitching (not supported yet).
* **Template Stitching**: An older stitching algorithm that provides poor stitching results for near-field scenes, but is fast and has low computational cost.
* **Dynamic Stitching**: Suitable for scenes containing motion or situations with rapid changes in movement.
* **Optical Flow Stitching**: Similar in function to dynamic stitching but optimized for higher accuracy.
* **AI Stitching**: Based on the optimization algorithm of Insta360's existing optical flow stitching technology, it provides better stitching effect. This stitching mode requires the configuration of the stitching model path (aiStitchPath). The model file is in the project file, `model_export_v22_d7c49187.mlmodelc`

```Objective-C
@property (nonatomic) NSString *aiStitchPath;
```

```objective-c
typedef NS_ENUM(NSInteger, INSOpticalFlowType) {
    // Dynamic Stitching
    INSOpticalFlowTypeDynamicStitch = 0,
    // Optical Flow Stitching
    INSOpticalFlowTypeDisflow = 1,
    // AI Stitching
    INSOpticalFlowTypeAiFlow = 2,
};
```

#### Stabilization mode（stabMode）

The Stabilization mode supports the following types in total, and there are two commonly used ones, enable stabilization and directional locking.
- Enable Stabilization: No matter how the camera is rotated, the camera image remains unchanged.
- Enable Direction Lock: The viewing angle always rotates with the camera
```objective-c
/**
 * Stabilization mode, default INSStabilizerStabModeZDirectional
 */
@property (nonatomic)INSStabilizerStabMode stabMode;
```

```objective-c
typedef NS_ENUM(NSInteger, INSStabilizerStabMode) {
    /// Turn off stabilization
    INSStabilizerStabModeOff = -1,
    /// Enable Stabilization
    INSStabilizerStabModeStill,
    /// Remove the Stabilization in the yaw direction, used for directional locking, view can only move arround z-axis (pan move)
    INSStabilizerStabModeZDirectional,
    /// Turn on Stabilization and turn on horizontal correction, that is, full-direcional.
    INSStabilizerStabModeFullDirectional,
    /// Uncalibrated horizon, i.e. free-footage, total free camera mode, camera can move through three axes
    INSStabilizerStabModeFreeFootage,
    /// Handling flip
    INSStabilizerStabModeFlipEffect,
    /// Turn on Stabilization and average posture correction, that is, relative-refine.
    INSStabilizerStabModeRelativeRefine,
    /// Turn on Stabilization and turn on horizontal correction, that is, absulute-refine.
    INSStabilizerStabModeAbsoluteRefine,
    /// Bullet time
    INSStabilizerStabModeBulletTime,
    /// Pano Fpv
    INSStabilizerStabModePanoFPV
};
```
#### Lens Guard

If lens guards are worn during shooting, this parameter needs to be configured according to the type. (Standard lens guards in the store are classified as A-grade, while Premium guards are classified as S-grade.)
```objective-c
/**
 * Lens Guard
 */
@property (nonatomic) INSOffsetConvertOptions protectType;
```

```objective-c
typedef NS_ENUM(NSInteger, INSProtectType) {
    // No Lens Guard
    INSProtectTypeNone = 0,
    // X3 S-grade
    INSProtectType_OneX3_Glass = 1,
    // X3 A-grade
    INSProtectType_OneX3_PlasticCement = 2,
    // X4 S-grade
    INSProtectType_OneX4_Glass = 3,
    // X4 A-grade
    INSProtectType_OneX4_PlasticCement = 4,
};
```
#### Chromatic Calibration（colorFusion）

```objective-c
/**
 * Chromatic Calibration fault: false
 */
@property (nonatomic) BOOL colorFusion;
```

The main reasons for color difference are as follows:

* Firstly, due to the fact that the two lenses are separated, their respective video exposures may not be consistent. When they are spliced together, there will be a noticeable brightness difference. This is because there are differences in parameter settings and photosensitive element performance during shooting between different lenses, resulting in different light intensities captured.

* Secondly, because the lighting on both sides of the lens is different and the camera exposure is different, sometimes there will be a significant brightness difference in the images taken by the front and rear lenses. This phenomenon is particularly evident in areas with large light aberration ratios, such as areas where strong light meets shadows, or in environments with large differences in indoor and outdoor light. In these cases, the camera finds it difficult to balance the light intensity in different areas, resulting in significant differences in brightness in the captured images.

It is precisely to solve such problems that chromatic calibration technology has been developed. It adjusts and optimizes the images captured by different lenses through a series of algorithms and image processing methods to reduce or eliminate brightness differences, making the spliced video look more natural and coherent.

#### Removing purple fringing（defringeConfig）

This interface has the function of eliminating specific phenomena. Specifically, it is specifically used to eliminate the purple edge phenomenon caused by lighting factors during the recording process. In actual recording scenes, the lighting conditions are complex and changeable. This interface can specifically solve the purple edge problem that occurs in two common lighting scenes. The first is the outdoor strong light scene. When recording in strong sunlight during the day, the outdoor high-intensity light is prone to cause the purple edge phenomenon. This interface can effectively eliminate the purple edge phenomenon in this scene. The second is the indoor lighting scene. The light emitted by different types of indoor lights such as incandescent lamps and fluorescent lamps may also cause the recorded image to appear purple edge. This interface can also eliminate the purple edge phenomenon in this indoor lighting scene. 

Usage: To use this algorithm, you need to configure the corresponding model, which is managed by a json file (`coreml_model_v2_scale6_scale4 json`). You need to copy these models and json files to the App. Please refer to the following code for parameter configuration

**Video：**

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

**Picture：**

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

#### Noise Reduction

In the field of video processing, multi-frame noise reduction technology is an important image processing method that can reduce or remove noise from video images. Noise affects image quality and viewing experience. Unlike single-frame noise reduction that only processes a single frame, multi-frame noise reduction uses redundant information from multiple frames before and after to more accurately distinguish image content from noise. It can retain image details while removing noise to obtain high-quality images. However, multi-frame noise reduction also has disadvantages. It needs to process multiple frames of information, and the calculation and analysis are complex, which consumes system performance, may cause the device to run slower and freeze, and will also slow down the video export speed, increasing the time cost.

```Objective-C
/**
* Noise Reduction
* This property is used to control whether to enable the multi-frame noise reduction function of the video. When set to YES, the noise reduction function is enabled; when set to NO, the noise reduction function is disabled.
*/
@property (nonatomic) BOOL enableDenoise;
```

#### Color Enhancement

```Objective-C
/**
* Color enhancement, default false closed
*/
@property (nonatomic) BOOL clolorPlus;
```

#### Progress and status callbacks

Once you start exporting, the protocol (INSRExporter2ManagerDelegate) pulls back the progress and status of the export in real time.

```objective-c
typedef NS_ENUM(NSInteger, INSExporter2State) {
    INSExporter2StateError = -1, // Indicates that an error occurred during the export process
    INSExporter2StateComplete = 0, // Indicates that the export was successfully completed
    INSExporter2StateCancel = 1, // Indicates that the user manually canceled the export operation
    INSExporter2StateInterrupt = 2, // Indicates that export was unexpectedly interrupted
    INSExporter2StateDisconnect = 3, // Indicates disconnection from the server
    INSExporter2StateInitError = 4, // Indicates that an error occurred during initialization
};
```

```objective-c
@protocol INSRExporter2ManagerDelegate  <NSObject>

- (void)exporter2Manager:(INSExporter2Manager *)manager progress:(float)progress;

- (void)exporter2Manager:(INSExporter2Manager *)manager state:(INSExporter2State)state error:(nullable NSError*)error;

// For panoramic event tracking detection (generally not used)
- (void)exporter2Manager:(INSExporter2Manager *)manager correctOffset:(NSString*)correctOffset errorNum:(int)errorNum totalNum:(int)totalNum clipIndex:(int)clipIndex type:(NSString*)type;
@end
```

#### Error code

Export error code.

```objective-c
typedef NS_ENUM(NSInteger, INSExporter2State) {
    INSExporter2StateError = -1, // Indicates that an error occurred during the export process
    INSExporter2StateComplete = 0, // Indicates that the export was successfully completed
    INSExporter2StateCancel = 1, // Indicates that the user manually canceled the export operation
    INSExporter2StateInterrupt = 2, // Indicates that export was unexpectedly interrupted
    INSExporter2StateDisconnect = 3, // Indicates disconnection from the server
    INSExporter2StateInitError = 4, // Indicates that an error occurred during initialization
};
```

## Export (image)

### Instructions for use

Image Export: `INSExportImageSimplify` class belongs to the `INSCoreMedia` framework. This class aims to simplify image export operations, providing a series of configurable properties and methods, supporting the export of `normal` and `HDR` images, allowing developers to customize the export process according to their needs.

#### Export normal image

```objective-c
- (NSError*)exportImageWithInputUrl:(nonnull NSURL *)inputUrl outputUrl:(nonnull NSURL *)outputUrl;
```

* **Function**: Export normal images, support remote or local images. Support remote and local

* **Parameter**：
  * `inputUrl`: The URL of the input image cannot be empty.
  * `outputUrl`: The URL of the output image cannot be empty.

* **Return value**: If an error occurs during the export process, return an `NSError` object; if the export is successful, return `nil`.

* **Example code**:

```objective-c
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

#### Export HDR image method (only supports local HDR images)

```objective-c
- (NSError*)exportHdrImageWithInputUrl:(nonnull NSArray<NSURL*> *)inputUrl outputUrl:(nonnull NSURL *)outputUrl;
```

* **Function**:
  * This feature is mainly used to export high dynamic range (HDR) images. It only supports local images, which means that the images must be stored on the local device for export operation. In addition, to ensure the quality and effect of export, the number of input images is required to be at least 3.

* **Parameters**:
  * `inputUrl`: This is an array of URLs for input images. The array cannot be empty and the number of elements in it must be greater than or equal to 3. These URLs point to local images that need to be exported through HDR.
  * `outputUrl`: Specify the URL of the output image. Similarly, this parameter cannot be empty and is used to determine the storage location of the exported HDR image.

* **Return value**:
  * During the export process, if any errors occur, an `NSError` object will be returned. This object contains detailed information about the error for developers to handle and debug.
  * If the export is successful and no errors occur, the return value will be `nil`, indicating that the operation is successfully completed.

* **Example code**:
```objective-c
INSExportImageSimplify *exporter = [[INSExportImageSimplify alloc] init];
NSArray<NSURL*> *inputUrls = @[[NSURL fileURLWithPath:@"path/to/input1.jpg"], [NSURL fileURLWithPath:@"path/to/input2.jpg"], [NSURL fileURLWithPath:@"path/to/input3.jpg"]];
NSURL *outputUrl = [NSURL fileURLWithPath:@"path/to/output_hdr.jpg"];
NSError *error = [exporter exportHdrImageWithInputUrl:inputUrls outputUrl:outputUrl];
if (error) {
    NSLog(@"导出失败：%@", error.localizedDescription);
} else {
    NSLog(@"导出成功");
}

//"导出失败" means "Export failed". "导出成功" means "Export successful".
```

#### Error code

```objective-c
typedef NS_ENUM(NSInteger, INSExportSimplifyError) {
    INSExportSimplifyErrorSuccess = 0, // Success
    INSExportSimplifyErrorInitFailed = 3001, // Export failed
    INSExportSimplifyErrorImageInitFailed = 4001, // Initialization failed
};
```

### Parameter description

Please refer to video parameters, image parameters are a subset of video parameters. Difference: There is no callback method for exporting images, and error messages are directly returned through the return value.

### Other features

#### LOG management

In the INSCoreMedia library, the management of logs can be completed with the help of INSRegisterLogCallback class. This class provides rich functions. Firstly, it supports flexible configuration of log output paths. Developers can output logs to specified files or storage locations according to their own project needs, which is convenient for centralized viewing and analysis of logs in the future. Secondly, this class also has the function of log level filtering. Different levels of logs, such as debugging level, information level, warning level, error level, etc., have different importance in different stages of project development and operation. By configuring filtering levels, only logs of specific levels and above can be output, thereby reducing unnecessary log information interference and improving the efficiency of development and debugging.

In addition, INSCoreMedia also supports registering callbacks. Users can customize log processing in the callback by registering the callback function. For example, log information can be printed to the Console in the callback, making it convenient for developers to view log content in real-time during debugging; logs can also be stored as log files for detailed tracing and analysis of historical logs, providing strong data support for project maintenance and optimization. 

**Configure log output file:**

```Objective-C
/**
     1.If the configuration log includes file output, the file path needs to be set.
     2.Set the output log level, logs greater than or equal to this level will be output.
 */
- (void)configLogKind:(InsLogKind)kind minLogLevel:(InsLogLevel)level optionalFullFilePath:(NSString * __nullable)fullFilePath;
```

**Registration Callback:**

```Objective-C
/**
  Custom log received callbacks
*/
- (void)registerLogCallBack:(void(^)(NSString *_Nonnull tag, InsLogLevel level, NSString * _Nonnull fileName, NSString * _Nonnull funcName, int line, NSString *_Nonnull message))block;
- (void)registerLogCallBackBmg:(void(^)(InsLogLevel level,NSString *_Nonnull message))block;
```

# FAQ

## Issues related to URI returned by HDR shooting

1. x4 and x5 support in-camera HDR fusion. After turning on in-camera fusion, HDR synthesis will be synthesized in the camera, and only one synthesized image will be generated after synthesis. 
2. Cameras x3 and earlier will return multiple URIs when shooting HDR images, but in later cameras such as x4 and x5, only one URI will be returned regardless of whether in-camera synthesis is turned on or not (not supported by the firmware end)
