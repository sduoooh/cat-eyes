## 这是什么？
[绮良良](https://bbs.mihoyo.com/ys/obc/content/6594/detail?bbs_presentation_style=no_header)猫箱急件状态下的眼睛动画demo，参照下图制作：      
![猫盒眼睛](./catbox.png)

## 目前实现了什么效果
- 基于web的demo
- 瞳孔跟随焦点移动，带动整个眼睛运动
- 随机眨眼
- 焦点通过鼠标位置控制
- 焦点通过摄像头获取手部位置进行控制（摄像头权限+PC前置摄像头），但没有根据图像信息判断摄像头方向，手机端估计有bug.

## todo
- 瞳孔跟随焦点移动，带动整个眼睛运动
- 根据深度值变化调整瞳孔大小
- 增加原定的位移过程中通过控制控制点相对移动产生的眼睛方向性变化
- 转译到flutter

## 一些记录
- 使用 google 的 mediapipe 库进行手部点位置获取。部署时同[Vite 的一个issue](https://github.com/vitejs/vite/issues/15851),打包后出现错误：
    ```
    '"Hands" is not a constructor'.   
    ```      
    据 [mediapipe 的一个 issue](https://github.com/google-ai-edge/mediapipe/issues/1976), 以下更改后可解决：
    ```
    import { Hands as _Hands } from '@mediapipe/hands';
    const Hands = _Hands || window.Hands;
    const hands = new Hands({ locateFile: (file) => { `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` } });
    ``` 

