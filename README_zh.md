# 简要

iOS SDK主要用于连接、设置和获取相机参数、控制相机进行拍照和录制、文件下载、固件升级和支持视频导出与图片导出等。

支持机型：X5, X4, X3, ONE X2, ONE X, ONE RS, ONE RS一英寸。

# 目录
* [INSCameraSDK](#inscamerasdk)
  * [环境准备](#环境准备)
  * [相机连接与相机状态](#相机连接与相机状态)
  * [拍摄模式](#拍摄模式)
  * [相机参数](#相机参数)
  * [拍照](#拍照)
  * [录制](#录制)
  * [预览](#预览)
  * [其他功能](#其他功能)
* [INSMediaSDK](#insmediasdk)
  * [导出（视频）](#导出视频)
  * [导出(图片)](#导出图片)


# INSCameraSDK

## 环境准备

### 通过Wi-Fi连接相机

- **启用相机 Wi-Fi 功能**

  在相机上开启 Wi-Fi 功能。

- **连接 Wi-Fi**

  在手机的 Wi-Fi 设置中，查找并选择对应的相机名称。

- **输入连接密码**

| 机型       | 密码获取方式                                       |
| -------- | -------------------------------------------- |
| X2       | 固定密码：`88888888`                              |
| X3/X4/X5 | 手动获取：相机屏幕 → 设置 → Wi-Fi → 显示密码<br>蓝牙获取：见1.2.3 |

### 通过蓝牙连接相机

通过蓝牙连接相机后，可以正常使用拍摄、录制等功能，但不支持实时预览和文件下载。如果您需要预览功能，可以在蓝牙配对成功后，通过调用相应的接口获取到WIFI的名称和密码，然后通过wifi连接

####  搜索蓝牙设备

调用以下方法扫描蓝牙设备

```python
INSBluetoothManager().scanCameras
```

#### 建立蓝牙连接

使用下面的接口建立蓝牙连接：

```c++
- (id)connectDevice:(INSBluetoothDevice *)device
         completion:(void (^)(NSError * _Nullable))completion;
```

#### 获取相机wifi账号和密码

蓝牙连接成功后，调用接口获取 Wi-Fi 信息。示例代码如下：

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

## 相机连接与相机状态

### 连接相机

通过 Wi-Fi 连接相机后，可调用 INSCameraSDK 中的接口建立连接。该接口采用单例模式，首次调用时会创建实例，后续调用则复用同一实例。

```c++
INSCameraManager.socket().setup()
```

### 断开相机

当需要断开与相机的连接时，请调用以下接口：

```c++
INSCameraManager.socket().shutdown()
```

### 判断相机连接状态

通过以下方式获取相机的连接状态：

```swift
INSCameraManager.socket().cameraState
```

返回状态中，若状态为 `INSCameraStateConnected` 则表示已成功建立连接。状态定义如下:

```c++
INSCameraManager.socket().cameraState

// 总共有以下几种状态
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

## 拍摄模式

本节主要介绍如何设置与获取相机的各项参数，包括拍摄模式、摄影参数等。参数类型主要分为数值型（如曝光 ISO、EV 值、快门速度等）和枚举型（如拍摄模式、曝光模式、分辨率等）。

###  获取当前拍摄模式

使用接口 `INSCameraManager.shared().commandManager.getOptionsWithTypes` 获取当前相机的拍摄模式。该方法需要传入以下参数：
* **optionTypes**
  指定需要获取哪些参数，每个参数对应一个枚举值（参见 `INSCameraOptionsType`），需要转换为 `NSNumber` 数组。
* **requestOptions**
  可配置请求的最大超时时长（默认 10 秒）。
* **completion**
  回调中返回：
  * `error`：为 `nil` 表示获取成功，否则包含错误原因。
  * `cameraOptions`：包含当前相机拍摄模式（拍摄模式为 `photoSubMode` 或 `videoSubMode`，具体枚举含义参见下文）。

拍摄模式枚举：
```typescript
// 拍照
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
// 视频录制
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

调用demo：
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

### 设置拍摄模式

通过方法：`INSCameraManager.shared().commandManager.setOptions`将指定模式配置给相机。在设置拍摄模式中有以下几个重要的点
* **INSCameraOptions**
  用于承载拍摄模式配置，主要包含以下两个属性：
  * `photoSubMode`：拍照模式
  * `videoSubMode`：录像模式
* **INSCameraOptionsType**
  用于指定 `INSCameraOptions` 中哪些参数需要生效：
  * 拍照模式：`INSCameraOptionsType::INSCameraOptionsTypePhotoSubMode`
  * 录制模式：`INSCameraOptionsType::INSCameraOptionsTypeVideoSubMode`

以下为示例代码（以设置图片拍摄模式为例）：

```c++
// 图片拍摄模式枚举
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

## 相机参数

相机参数通常分为两类，第一种为具体的数值，如曝光iso，ev值，快门速度等。第二种为枚举值，如拍摄模式，曝光模式等。对于前者，能够直接向 `INSPhotographyOptions` 中的参数赋值，并发送至相机。对于后者，则需查阅表中对应枚举所表示的含义，再将对应的枚举值赋值给`INSPhotographyOptions`。如下所示为分辨率与其对应枚举值的关系。其他参数可通过SDK工程中的`“common_camera_setting_proto.json”`文件查看。

### 获取相机参数

获取相机通过方法：`getPhotographyOptions`获取。需要的主要入参有三个`functionMode`和`optionTypes`，以及一个闭包回调。

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
* `functionMode`：表示当前的拍摄模式

* `optionTypes`：每个参数对应一个枚举值，
  * 如分辨率对应：INSPhotographyOptionsTypePhotoSize = 40
  * 白平衡对应：INSPhotographyOptionsTypeWhiteBalance = 13

* 闭包会返回一个`NSError`和`INSPhotographyOptions`
  * `NSError`为`nil`表示成功，否则会包含失败原因
  * `INSPhotographyOptions`中包含当前相机参数的具体值，如分辨率，白平衡等



示例代码如下：获取当前相机白平衡的值
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


### 设置相机参数

给相机设置参数需要通过方法：`INSCameraManager.shared().commandManager.setPhotographyOptions`，需要的主要入参主要有以下几个参数。

* `INSPhotographyOptions`：存储被配置的参数

* `INSPhotographyOptionsType`：表示哪些参数需要被配置（每个参数对应一个枚举）

* `INSCameraFunctionMode`：当前的拍摄模式，当前拍摄模式。

  * 相机设置需要的拍摄模式`INSCameraFunctionMode`和从相机拿到的拍摄模式`INSPhotoSubMode`或`INSVideoSubMode`不能直接转换，对应关系参考工程文件中的类：`HUMCaptureMode`。通过`HUMCaptureMode`将相机当前模式INSPhotoSubMode或INSVideoSubMode转成统一的`INSCameraFunctionMode`

* 闭包回调：返回是否获取成功

> 所有参数类型只能设置为相机屏幕上支持的参数，不能随意设置

示例代码如下：设置指定白平衡
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

### 数值类型

#### 白平衡

* `INSPhotographyOptions`中的属性为：

```objective-c
@property (nonatomic) uint32_t whiteBalanceValue;
```

* 在`INSPhotographyOptionsType`中对应的枚举为：

```objective-c
INSPhotographyOptionsTypeWhiteBalance = 13
```

* 假设当前拍摄模式`INSCameraFunctionMode`为6表示普通拍照。

#### ExposureBias（仅自动曝光支持"AUTO）

> 仅在曝光模式为"AUTO" "FULL\_AUTO"下支持调节

* `INSPhotographyOptions`中的属性为：exposureBias

```objective-c
@property (nonatomic) float exposureBias;
```

* `INSPhotographyOptionsType`中对应的枚举为：

```objective-c
INSPhotographyOptionsTypeExposureBias = 7
```

#### ISO，shutterSpeed

> 仅在曝光模式为"MANUAL"和"ISO\_PRIORITY"两种模式下支持调节

* `INSPhotographyOptions`中的分别属性为：stillExposure.iso，stillExposure.shutterSpeed

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

* `INSPhotographyOptionsType`中对应的枚举为：

```objective-c
INSPhotographyOptionsTypeStillExposureOptions = 20
```

### 枚举和Bool

#### 图片分辨率

* `INSPhotographyOptions`中的属性为：photoSizeForJson

* photoSizeForJson = 17，表示x4相机的分辨率为8192 x 6144。

* 在`INSPhotographyOptionsType`中对应的枚举为：

```objective-c
INSPhotographyOptionsTypePhotoSize = 40
```

* 假设当前拍摄模式`INSCameraFunctionMode`为6表示普通拍照。

代码示例：

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

分辨率与枚举值的对应关系
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

#### 曝光模式

* `INSPhotographyOptions`中的属性为：

```objective-c
@property (nullable, nonatomic) INSCameraExposureOptions *stillExposure;


@interface INSCameraExposureOptions : NSObject

@property (nonatomic) uint8_t program; // 曝光模式

@property (nonatomic) NSUInteger iso;

#if TARGET_OS_IOS
@property (nonatomic) CMTime shutterSpeed;
#endif

@end
```

* `INSPhotographyOptionsType`中对应的枚举为：

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

> 支持同时设置多个参数给相机，如下

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

> 注意：X5默认打开pureshot模式，且不支持关闭

* `INSPhotographyOptions`中的属性为：

```objective-c
@property (nonatomic) uint8_t rawCaptureType;
```

* 在`INSPhotographyOptionsType`中对应的枚举为：

```objective-c
INSPhotographyOptionsTypeRawCaptureType = 25,
```

* 枚举类型：

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

#### 机内拼接（Bool）

在拍摄图片时，在相机内部拼接全景图像，无需再外部拼接素材。仅x4及x4之后的相机支持。

```swift
var typeArray = [INSCameraOptionsType]()
let cameraOptions = INSCameraOptions()

// 打开为true， 关闭为false
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


## 拍照

通过“`INSCameraManager.shared().commandManager.takePicture`”方法向相机下达拍摄指令，若 error 为 nil，则表示拍摄成功，反之则包含错误信息。optionInfo 中涵盖照片的远程访问地址，可进行远程导出（无需下载至本地）。

```bash
INSCameraManager.shared().commandManager.takePicture(with: nil, completion: { (error, optionInfo) in
    
    print("end takePicture \(Date())")     
    guard let uri = optionInfo?.uri else {
        return
    }
    print("Take Picture Url:\(uri)")\
})
```

## 录制

开始录制：
```python
INSCameraManager.shared().commandManager.startCapture(with: nil, completion: { (err) in
    if let err = err {
        self.showAlert(row.tag!, "\(String(describing: err))")
        return
    }
})
```

停止录制：
```python
INSCameraManager.shared().commandManager.stopCapture(with: nil, completion: { (err, info) in
    if let err = err {
        self.showAlert(row.tag!, "\(String(describing: err))")
        return
    }
})
```

##  预览

预览必须在和相机建立连接之后才行，有以下几个核心类，具体参考demo：CameraConfigByJsonController

* INSCameraSessionPlayer

  * 绑定ios端的bounds

  * 实现协议：INSCameraSessionPlayerDelegate

  * 实现协议：INSCameraSessionPlayerDataSource，建议一定实现下面两个协议：

    配置offset信息：updateOffsetToPlayer

    配置防抖信息：updateStabilizerParamToPlayer

* INSCameraMediaSession

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



##  其他功能

### 固件升级

通过调用updateFirwareWithOptions接口主动升级。INSCameraWriteFileOptions

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



### 激活

Appid和secret需要向Insta360申请：

```swift
guard let serialNumber = INSCameraManager.shared().currentCamera?.serialNumber else {
    self?.showAlert("提示", "请先连接相机")
    return
}
let commandManager = INSCameraManager.shared().commandManager

// 需要申请，填入真实的appid和secret
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

###  获取SD卡状态

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

### 获取当前电量

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

### 获取激活时间

```swift
let optionTypes = [
         NSNumber(value: INSCameraOptionsType.activateTime.rawValue),
      ];
INSCameraManager.shared().commandManager.getOptionsWithTypes(optionTypes) { (err, options, successTypes) in
    guard let options = options else {
        self.showAlert("get options", String(describing: err))
        return
    }
            // 激活时间
    let formatter = DateFormatter()
    formatter.dateFormat = "yyyy-MM-dd HH:mm:ss.SSS"
    let date = Date(timeIntervalSince1970: Double(options.activateTime / 1000))
    print("date :\(date)")
    
}
```

### 静音开关

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

### 序列号

与相机建立连接之后即可调用

```swift
INSCameraManager.shared().currentCamera?.serialNumber
```

### 固件版本号

```swift
INSCameraManager.shared().currentCamera?.firmwareRevision
```

***

### LOG管理

在INSCameraSDK里，其所有产生的日志（log）都会借助`INSCameraSDKLoggerProtocol`协议，以回调的形式传递到上层。对于上层而言，接收到这些日志之后，既能够选择将日志进行打印输出，方便开发人员在调试阶段查看SDK运行过程中的相关信息，也可以对日志进行存储操作，以便后续进行数据分析或者问题排查。若想要了解更为详细的操作和配置，可以具体参考`CameraConfigByJsonController`。 

`INSCameraSDKLogger`以一个单例作为`INSCameraSDKLoggerProtocol`的载体，需要配置`logDelegate`属性来承载`INSCameraSDKLoggerProtocol`的回调

```objective-c
@protocol INSCameraSDKLoggerProtocol

- (void)logError:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logWarning:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logInfo:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;
- (void)logDebug:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;

- (void)logCrash:(NSString *)message filePath:(NSString *)filePath funcName:(NSString *)funcName lineNum:(NSInteger)lineNum;

@end
```

### 错误码

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

## 导出（视频）

视频导出：为tob用户专门提供了一套简化的导出方法`INSExportSimplify` ，位于 `INSCoreMedia` 该类主要用于简化视频导出的操作，提供了一系列可配置的属性和方法，方便开发者根据需求定制导出过程。`（仅支持导出视频）`。

### 使用说明

这种方法可以保证基本的导出效果，所有导出所使用的参数都是默认值。用户可以根据特定需求对相关参数进行配置。

#### 初始化方法

```objective-c
-(nonnull instancetype)initWithURLs:(nonnull NSArray<NSURL>)urls outputUrl:(nonnull NSURL*)outputUrl;
```

* **功能**：初始化 `INSExportSimplify` 对象，用于指定输入视频文件的 URL 数组和输出视频文件的 URL。

* **参数**：
  * `urls`：输入视频文件的 URL 数组，不能为空。
  * `outputUrl`：输出视频文件的 URL，不能为空。

* **返回值**：返回一个 `INSExportSimplify` 对象。

* **示例代码**：

```objective-c
NSArray<NSURL*> *inputUrls = @[[NSURL fileURLWithPath:@"path/to/input1.mp4"], [NSURL fileURLWithPath:@"path/to/input2.mp4"]];
NSURL *outputUrl = [NSURL fileURLWithPath:@"path/to/output.mp4"];
INSExportSimplify *exporter = [[INSExportSimplify alloc] initWithURLs:inputUrls outputUrl:outputUrl];
```

#### 开始导出

```objective-c
-(nullable NSError*)start;
```

#### 取消导出

```objective-c
-(void)cancel;
```

#### 销毁导出

取消导出，导出完成之后，一定要主动调用，否则可能导致crash

```objective-c
-(void)shutDown;
```

### 参数说明

#### 导出分辨率

该接口用于输出图像分辨率，(宽高比必须为2:1)

```objective-c
/**
 * 默认宽高： 1920 * 960 （导出画面比例需要为2:1）
 */
@property (nonatomic) int width;
@property (nonatomic) int height;
```

#### 拼接模式（opticalFlowType）

总共有四种拼接模式，分别表示，模版拼接，动态拼接，光流拼接，Ai拼接（暂不支持）。
- 模板拼接：比较老的拼接算法，对近景拼接效果不好，但是速度快，性能消耗低
- 动态拼接：适合包含近景的场景，或者有运动和快速变化的情况
- 光流拼接：使用场景和动态拼接相同
- AI 拼接：基于影石 Insta360 现有的光流拼接技术的优化算法，提供更优的拼接效果
```objective-c
typedef NS_ENUM(NSInteger, INSOpticalFlowType) {
    // 动态拼接
    INSOpticalFlowTypeDynamicStitch = 0,
    // 光流拼接
    INSOpticalFlowTypeDisflow = 1,
    // AI拼接
    INSOpticalFlowTypeAiFlow = 2,
};
```

#### 防抖模式（stabMode）

防抖模式总共支持以下几种，常用的有两种，全防和方向锁定
- 全防（Still）：无论怎么转动相机，相机画面始终保持不动()
- 方向锁定（FullDirectional）：视角始终随着相机的转动而转动

```objective-c
/**
 * 防抖模式，默认 INSStabilizerStabModeZDirectional
 */
@property (nonatomic)INSStabilizerStabMode stabMode;
```

```objective-c
typedef NS_ENUM(NSInteger, INSStabilizerStabMode) {
    /// 关闭防抖
    INSStabilizerStabModeOff = -1,
    /// 全防
    INSStabilizerStabModeStill,
    /// 移除yaw方向的防抖, 用于directional lock, view can only move arround z-axis (pan move)
    INSStabilizerStabModeZDirectional,
    /// 开防抖、开水平矫正，即full-direcional
    INSStabilizerStabModeFullDirectional,
    ///不校准地平线, 即free-footage, total free camera mode, camera can move through three axis
    INSStabilizerStabModeFreeFootage,
    ///  //处理翻转
    INSStabilizerStabModeFlipEffect,
    ///开防抖、平均姿态矫正，即relative-refine
    INSStabilizerStabModeRelativeRefine,
    ///开防抖、开水平矫正，即absulute-refine
    INSStabilizerStabModeAbsoluteRefine,
    ///子弹时间
    INSStabilizerStabModeBulletTime,
    /// Pano Fpv
    INSStabilizerStabModePanoFPV
};
```

#### 保护镜

如果拍摄时佩戴保护镜，需要根据类型配置该参数。（商城中的标准保护镜为A级，高级保护镜为S级）

```objective-c
/**
 * 保护镜
 */
@property (nonatomic) INSOffsetConvertOptions protectType;
```

```objective-c
typedef NS_OPTIONS(NSUInteger, INSOffsetConvertOptions) {
    INSOffsetConvertOptionNone                   = 0,
    INSOffsetConvertOptionEnableWaterProof       = 1 << 0, // ONE X 保护镜
    INSOffsetConvertOptionEnableDivingAir        = 1 << 1, // 潜水壳（水上）
    INSOffsetConvertOptionEnableDivingWater      = 1 << 2, // 潜水壳（水下）
    INSOffsetConvertOptionEnableDivingAirV2      = 1 << 3, // 新版潜水壳（水上）
    INSOffsetConvertOptionEnableDivingWaterV2    = 1 << 4, // 新版潜水壳（水下）
    INSOffsetConvertOptionEnableBuckleShell      = 1 << 5, // 卡扣式弧面镜
    INSOffsetConvertOptionEnableAdhesiveShell    = 1 << 6, // 黏贴式球面镜
    INSOffsetConvertOptionEnableGlassShell       = 1 << 7, // 玻璃保护镜(S)
    INSOffsetConvertOptionEnablePlasticCement    = 1 << 8, // 塑胶保护镜(A)
    INSOffsetConvertOptionEnableAverageShell     = 1 << 9  // A/S平均
};
```

#### 消色差（colorFusion）

```objective-c
/**
 * 消色差: 默认false
 */
@property (nonatomic) BOOL colorFusion;
```

产生色差的原因主要有以下两点：

* 首先，由于两个镜头是分开的，其各自得出的视频曝光可能不太一致。当把它们拼接在一起的时候，会有比较明显的亮度差。这是因为不同镜头在拍摄时的参数设置、感光元件的性能等方面存在差异，从而导致所捕捉到的光线强度有所不同。

* 其次，因为镜头两边的光照不一样，相机曝光不同，有时候前后镜头拍出来的画面也会有明显的亮度差。这种现象在光差比大的地方尤其明显，比如在强光与阴影交界的区域，或者是在室内外光线差异较大的环境中。在这些情况下，相机难以平衡不同区域的光线强度，从而使得拍摄出的画面在亮度上出现较大的差别。

正是为了解决此类问题，消色差技术得以开发。它通过一系列的算法和图像处理手段，对不同镜头拍摄的画面进行调整和优化，以减少或消除亮度差，从而使拼接后的视频看起来更加自然和连贯。

#### 去紫边（defringeConfig）

该接口具备消除特定现象的功能，具体而言，其专门用于消除录制过程中因光照因素产生的紫边现象。在实际录制场景中，光照情况复杂多变，该接口能够针对性地解决两类常见光照场景下出现的紫边问题。其一为室外强光场景，在阳光强烈的白天进行录制时，室外高强度光线极易引发紫边现象，此接口可有效消除该场景下的紫边；其二为室内灯光场景，室内诸如白炽灯、荧光灯等不同类型的灯光所发出的光线，也可能致使录制画面出现紫边，该接口同样能够对这种室内灯光场景下的紫边现象进行消除处理。 

使用方法：使用该算法，需要配置对应模型，该模型通过一个json文件来管理（coreml_model_v2_scale6_scale4.json）。需要将这些模型和json文件拷贝到App内。参数配置参考下列代码

视频

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

图片

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

#### 进度和状态回调

**开始导出后**，该协议（INSRExporter2ManagerDelegate）会实时回调导出的进度和状态。

```objective-c
typedef NS_ENUM(NSInteger, INSExporter2State) {
    INSExporter2StateError = -1, // 表示导出过程中出现错误
    INSExporter2StateComplete = 0, // 表示导出成功完成
    INSExporter2StateCancel = 1, // 表示用户手动取消了导出操作
    INSExporter2StateInterrupt = 2, // 表示导出被意外中断
    INSExporter2StateDisconnect = 3, // 表示与服务器断开连接
    INSExporter2StateInitError = 4, // 表示初始化时出现错误
};
```

```objective-c
@protocol INSRExporter2ManagerDelegate  <NSObject>

- (void)exporter2Manager:(INSExporter2Manager *)manager progress:(float)progress;

- (void)exporter2Manager:(INSExporter2Manager *)manager state:(INSExporter2State)state error:(nullable NSError*)error;

// 用于全景埋点检测（一般用不到）
- (void)exporter2Manager:(INSExporter2Manager *)manager correctOffset:(NSString*)correctOffset errorNum:(int)errorNum totalNum:(int)totalNum clipIndex:(int)clipIndex type:(NSString*)type;
@end
```

#### 错误码

导出错误码

```Objective-C
typedef NS_ENUM(NSInteger, INSExporter2State) {
    INSExporter2StateError = -1, // 表示导出过程中出现错误
    INSExporter2StateComplete = 0, // 表示导出成功完成
    INSExporter2StateCancel = 1, // 表示用户手动取消了导出操作
    INSExporter2StateInterrupt = 2, // 表示导出被意外中断
    INSExporter2StateDisconnect = 3, // 表示与服务器断开连接
    INSExporter2StateInitError = 4, // 表示初始化时出现错误
};
```

## 导出(图片)

### 使用说明

图片导出：`INSExportImageSimplify` 类属于 `INSCoreMedia` 框架。该类旨在简化图像导出操作，提供了一系列可配置属性和方法，支持导出`普通图像`和 `HDR 图像`，允许开发者根据需求定制导出过程。

#### 导出普通图像

```objective-c
- (NSError*)exportImageWithInputUrl:(nonnull NSURL *)inputUrl outputUrl:(nonnull NSURL *)outputUrl;
```

* **功能**：导出普通图像，支持远程或本地图片。支持远程和本地

* **参数**：
  * `inputUrl`：输入图像的 URL，不能为空。
  * `outputUrl`：输出图像的 URL，不能为空。

* **返回值**：如果导出过程中出现错误，返回一个 `NSError` 对象；如果导出成功，返回 `nil`。

* **示例代码**：

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

#### 导出 HDR 图像方法（仅支持本地HDR图片）

```objective-c
- (NSError*)exportHdrImageWithInputUrl:(nonnull NSArray<NSURL*> *)inputUrl outputUrl:(nonnull NSURL *)outputUrl;
```

* **功能**：
  * 该功能主要用于导出高动态范围（HDR）图像。它仅支持本地图像，这意味着图像必须存储在本地设备上才能进行导出操作。此外，为了确保导出的质量和效果，要求输入的图像数量至少为 3 张。

* **参数**：
  * `inputUrl`：这是一个输入图像的 URL 数组。该数组不能为空，且其中的元素数量必须大于等于 3。这些 URL 指向的是需要进行 HDR 导出的本地图像。
  * `outputUrl`：指定输出图像的 URL。同样，该参数也不能为空，它用于确定导出后的 HDR 图像的存储位置。

* **返回值**：
  * 在导出过程中，如果出现任何错误，将会返回一个`NSError`对象。这个对象包含了有关错误的详细信息，以便开发者进行错误处理和调试。
  * 如果导出成功，没有任何错误发生，那么返回值将为`nil`，表示操作顺利完成。

* **示例代码**：
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
```

#### 错误码

```objective-c
typedef NS_ENUM(NSInteger, INSExportSimplifyError) {
    INSExportSimplifyErrorSuccess = 0,// 成功
    INSExportSimplifyErrorInitFailed = 3001, // 导出失败
    INSExportSimplifyErrorImageInitFailed = 4001, // 初始化失败
};
```

### 参数说明

参考视频参数，图片参数是视频参数的子集。区别点：图片导出没有回调方法，直接通过返回值返回错误信息。

### 其他功能

#### LOG管理

在 INSCoreMedia 这个库当中，关于日志（log）的管理工作能够借助 INSRegisterLogCallback 类来完成。该类提供了颇为丰富的功能。其一，它支持对日志输出路径进行灵活配置。开发者可以依据自身项目的需求，将日志输出到指定的文件或者存储位置，这样便于后续对日志进行集中查看与分析。其二，该类还具备日志等级过滤的功能。不同等级的日志，例如调试级、信息级、警告级、错误级等，在项目开发和运行的不同阶段有不同的重要性。通过配置过滤等级，可以只输出特定等级及以上的日志，从而减少不必要的日志信息干扰，提高开发和调试的效率。

除此之外，INSCoreMedia 还支持注册回调。用户能够通过注册回调函数，在回调中实现对日志的自定义处理。比如，可以在回调中将日志信息打印到控制台，方便开发者在调试过程中实时查看日志内容；也可以将日志存储成 log 文件，便于后续对历史日志进行详细的追溯和分析，为项目的维护和优化提供有力的数据支持。 

配置log输出文件：

```Objective-C
/**
     1, 如果配置log包含文件输出，需要设置文件路径
     2, 设置输出log级别，大于等于该级别的log将会输出
 */
- (void)configLogKind:(InsLogKind)kind minLogLevel:(InsLogLevel)level optionalFullFilePath:(NSString * __nullable)fullFilePath;
```

注册回调

```objective-c
/**
  注册Custom log接收的回调
*/
- (void)registerLogCallBack:(void(^)(NSString *_Nonnull tag, InsLogLevel level, NSString * _Nonnull fileName, NSString * _Nonnull funcName, int line, NSString *_Nonnull message))block;
- (void)registerLogCallBackBmg:(void(^)(InsLogLevel level,NSString *_Nonnull message))block;
```





