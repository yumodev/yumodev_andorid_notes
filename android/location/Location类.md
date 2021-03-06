# Location类

[Android Developers Location](https://developer.android.com/reference/android/location/Location)

A data class representing a geographic location.
Location类是描述位置信息的类

A location can consist of a latitude, longitude, timestamp, and other information such as bearing, altitude and velocity.
包含纬度，经度，时间戳和其他信息，比如方位、高度和速度等信息

All locations generated by the LocationManager are guaranteed to have a valid latitude, longitude, and timestamp (both UTC time and elapsed real-time since boot), all other parameters are optional.
所有由LocationManager类产生的位置信息，都包含有效的经纬度和时间戳(启动的UTC时间和经过的实时时间)，其他的参数都是可选的。

## 属性和方法

方法和属性 | 含义 
------| ------
double getLatitude() | 纬度
double getLongitude() | 经度
String getProvider() | 返回适配器
float getSpeed() | 返回速度 m/s
float getSpeedAccuracyMetersPerSecond() | 速度精度
float getAccuracy() | 返回该位置的水平精度、径向 单位：米
double getAltitude() | 返回海拔
float getBearing() | 返回区位方向。
float getBearingAccuracyDegrees() | 方位精度
long getElapsedRealtimeNanos() | ...
long getTime() | ...
float getVerticalAccuracyMeters() | 获取到这个位置的垂直精度
boolean isFromMockProvider() | 
float bearingTo() | 





