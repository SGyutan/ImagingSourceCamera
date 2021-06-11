# ARGOの産業用UVCカメラ（ImagingSourceCamera）のインストールと動作テスト

#### はじめに

仕事でUVCカメラを使うことになり、そのインストールと動作確認の覚えとして記載します。なお下記のHP手順でインストール行いましたが、一部うまくいかなかったので、修正したものを乗せています。

参考：[AnacondaでTISカメラをPythonで動作させる方法](https://www.argocorp.com/UVC_camera/Windows_Python_Anaconda_operating.html)

[The Imaging Source社のカメラをPythonから動かす](https://raccoon15.hatenablog.com/entry/2019/04/21/000428)


#### 動作環境

win10 Pro

Anaconda

Anacondaについては、ライセンスの変更が話題になっています。ライセンスを確認してください。

Python=3.7

openCV=4.2

上記のHPを参考に

（1）下記の**「4-2）IC Imaging Control .NET Component for C#, VB.NET projects ソフトウェア
開発キット（SDK）」**をインストール
https://www.argocorp.com/software/DL/tis/index.html#tab4    

（2）下記のGithubで赤枠の**「Download ZIP」**をクリックし、ダウンロードします
https://github.com/TheImagingSource/IC-Imaging-Control-Samples      

このホルダーにはいろいろな開発言語がありますが、使うのはPythonだけです。

Python\Open Camera, Grab Image to OpenCVのホルダーの中のプログラムを以下の様に整理します。

dllsというホルダーを作ります。その中に以下のファイルを移動します。

```
Open Camera, Grab Image to OpenCV
|-----dlls
|      |----tisgrabber.py <---一部修正します
|      |----TIS_UDSHL11.dll
|      |----TIS_UDSHL11_x64.dll
|      |----tisgrabber.dll
|      |----tisgrabber_x64.dll 
|      |----tisgrabber.h
|      |----TISGrabberGlobalDefs.h
|
|---tis-OpenCV.py(Original)
|---callback.py(Original)
|---callback-image-processing.py(Original)
|---Using-Y16.py(Original)
|---tis-OpenCV_DMK33.py(New File) <---tis-OpenCV.pyをもとに新しいプログラムを作成します
```

### tisgrabber.pyの修正（dllsホルダーに移動）

```Python
from enum import Enum

import ctypes as C
import sys
import numpy as np

import os
os.environ['PATH'] = os.path.dirname(__file__) + ';' + os.environ['PATH']

```

の様に書き換えます。

プログラムの初めから

```python
import os
os.environ['PATH'] = os.path.dirname(__file__) + ';' + os.environ['PATH']
```

を追記します。

参考：[Python ctypes：loading DLL相対パスから](https://www.it-swarm-ja.com/ja/python/python-ctypes%EF%BC%9Aloading-dll%E7%9B%B8%E5%AF%BE%E3%83%91%E3%82%B9%E3%81%8B%E3%82%89/969732847/)

なお、下の方にある　 TIS_CAM（）　Classのメソッドsを下記の様に修正します。

```python
#修正前
class TIS_CAM(object):
	・・・
                                  
        def s(self,strin):
            if sys.version[0] == "2":
                return strin
            if type(strin) == "byte":
                return strin
            return strin.encode("utf-8")
```

`if type(strin) is bytes:`

に書き換えます。

型の判定は、`type()`または、`isinstance()`で判定します。また`==`ではなく`in` で判定します。

[Pythonで型を取得・判定するtype関数, isinstance関数](https://note.nkmk.me/python-type-isinstance/)

```Python
#修正後
class TIS_CAM(object):
・・・
                        
        def s(self,strin):
            if sys.version[0] == "2":
                return strin
            if type(strin) is bytes:
                return strin
            return strin.encode("utf-8")
```

この修正を行っておくと、

例えば、tis-OpenCV.pyでカメラのリストを取得します。Devices[]はBytes型になります。

```Python
Devices = Camera.GetDevices()
for i in range(len( Devices )):
    print( str(i) + " : " + str(Devices[i]))
    
# Open a device with hard coded unique name:
Camera.open(Devices[0])
```

そこで、このデバイスリストからカメラを指定すると（例えばDevices[0]）TIS_CAMのメソッドのsが呼ばれます。先ほどの修正をしていないとエラーが起こります。


デモプログラムのひな型のtis-OpenCV.pyのコピーを作成します。

私の場合は、使っているカメラがDMK33 UX273なのでtis-OpenCV_DMK33.pyとします。

tis-OpenCV_DMK33.pyの中のプログラムを書き換えます。

tis-OpenCV_DMK33.py

```python
import sys
# tisgrabberを参照できるように指定しておきます。
sys.path.append('./dlls')

import ctypes as C
import tisgrabber as IC
import cv2
import numpy as np


lWidth=C.c_long()
lHeight= C.c_long()
iBitsPerPixel=C.c_int()
COLORFORMAT=C.c_int()


# Create the camera object.
Camera = IC.TIS_CAM()

# List availabe devices as uniqe names. This is a combination of camera name and serial number
Devices = Camera.GetDevices()
for i in range(len( Devices )):
    print( str(i) + " : " + str(Devices[i]))

# Open a device with hard coded unique name:
# Camera.open("DMK 33UX273 5120007")

# tisgrabberのTISクラスの修正をした場合
# カメラが1台接続されていれば自動で選ばれます。
Camera.open(Devices[0])
# tisgrabberのTISクラスの修正をしてない場合はByteをStringにします。
# Camera.open(Devices[0].decode())

# Camera.ShowDeviceSelectionDialog()

if Camera.IsDevValid() == 1:
    #cv2.namedWindow('Window', cv2.cv.CV_WINDOW_NORMAL)
    # print( 'Press ctrl-c to stop' )
    print( 'Press q to stop' )

    # Set a video format
    #Camera.SetVideoFormat("RGB32 (640x480)")
    
    #Set a frame rate of 30 frames per second
    #Camera.SetFrameRate( 30.0 )
    
    # Start the live video stream, but show no own live video window. We will use OpenCV for this.
    # Camera.StartLive(1)   
    # if 1-> show original window
    # if 0 -> no show window
    Camera.StartLive(0) 
    
    # Set some properties
    # Exposure time
    ExposureAuto=[1]
    
    Camera.GetPropertySwitch("Exposure","Auto",ExposureAuto)
    print("Exposure auto : ", ExposureAuto[0])

    
    # In order to set a fixed exposure time, the Exposure Automatic must be disabled first.
    # Using the IC Imaging Control VCD Property Inspector, we know, the item is "Exposure", the
    # element is "Auto" and the interface is "Switch". Therefore we use for disabling:
    Camera.SetPropertySwitch("Exposure","Auto",0)
    # "0" is off, "1" is on.

    ExposureTime=[0]
    Camera.GetPropertyAbsoluteValue("Exposure","Value",ExposureTime)
    print("Exposure time abs: ", ExposureTime[0])

    
    # Set an absolute exposure time, given in fractions of seconds. 0.0303 is 1/30 second:
    Camera.SetPropertyAbsoluteValue("Exposure","Value",0.0303)

    # Proceed with Gain, since we have gain automatic, disable first. Then set values.
    Gainauto=[0]
    Camera.GetPropertySwitch("Gain","Auto",Gainauto)
    print("Gain auto : ", Gainauto[0])
    
    Camera.SetPropertySwitch("Gain","Auto",0)
    Camera.SetPropertyValue("Gain","Value",10)

    WhiteBalanceAuto=[0]
    # Same goes with white balance. We make a complete red image:
    Camera.SetPropertySwitch("WhiteBalance","Auto",1)
    Camera.GetPropertySwitch("WhiteBalance","Auto",WhiteBalanceAuto)
    print("WB auto : ", WhiteBalanceAuto[0])

    Camera.SetPropertySwitch("WhiteBalance","Auto",0)
    Camera.GetPropertySwitch("WhiteBalance","Auto",WhiteBalanceAuto)
    print("WB auto : ", WhiteBalanceAuto[0])
    
    Camera.SetPropertyValue("WhiteBalance","White Balance Red",64)
    Camera.SetPropertyValue("WhiteBalance","White Balance Green",64)
    Camera.SetPropertyValue("WhiteBalance","White Balance Blue",64)
    
    # try:
    while ( True ):
        # Snap an image
        Camera.SnapImage()
        # Get the image
        image = Camera.GetImage()
        # Apply some OpenCV function on this image
        # The obtained image is converted to mirror-inverted it
        image = cv2.flip(image,0)
        # image = cv2.erode(image,np.ones((11, 11)))

        
        # # 読み込んだ画像の高さと幅を取得
        height = image.shape[0]
        width = image.shape[1]

        # # 画像のサイズを変更
        # # 第一引数：サイズを変更する画像
        # # 第二引数：変更後の幅
        # # 第三引数：変更後の高さ
        resized_img = cv2.resize(image,(int(width), int(height)))
        # ウィンドウの表示形式の設定
        # 第一引数：ウィンドウを識別するための名前
        # 第二引数：ウィンドウの表示形式
        # cv2.WINDOW_AUTOSIZE：デフォルト。ウィンドウ固定表示
        # cv2.WINDOW_NORMAL：ウィンドウのサイズを変更可能にする
        cv2.namedWindow('Window', cv2.WINDOW_NORMAL)
        # resized_img = cv2.flip(resized_img, 1)
        cv2.imshow('Window', resized_img)
        # cv2.waitKey(10)
        
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
           
    # except KeyboardInterrupt:
    Camera.StopLive()    
    cv2.destroyWindow('Window')

    
else:
    print( "No device selected")import sys
```

Camera.StartLive(0) にして、表示はOpenCV側に任せます。そのほか下記を参考にOpenCV風に書き換えています。

参考：[カメラから動画を撮影する](http://labs.eecs.tottori-u.ac.jp/sd/Member/oyamada/OpenCV/html/py_tutorials/py_gui/py_video_display/py_video_display.html#display-video)
