
申明：本案例中的思路和技术仅用于学习交流。请勿用于非法行为。


一、训练模型


详细训练步骤和导出模型参考 [滑块验证码识别模型训练](https://github.com)


二、模型试用


通过YoloDotNet运行模型，计算出滑块缺口位置后用RESTful格式的接口返回坐标给其它应用调用。YoloDotNet案例参考 [物体检测框架YoloDotNet初体验](https://github.com):[milou加速器](https://xinminxuehui.org)。


主要步骤：


1\. 创建webapi(c\#)项目




```
# -n:指定项目名称为slider_apidotnet new webapi -n slider_api# 切换到项目目录下cd slider_api# 添加两项依赖dotnet add package System.Drawing.Commondotnet add package YoloDotNet
```


2\. 创建控制器，用于计算滑块位置并返回。几个需要注意的点：


* 创建Yolo模型时OnnxModel参数应设置为第一步中训练好的模型文件。
* 模型识别结果是一个BoundingBox对象，不是坐标位置，需要手动处理。BoundingBox类型为SKRectI，可以理解为一个矩形。观察BoundingBox中的属性值有Left、Right、Top、Bottom、Width等。那么缺口在背景图中的中心点的X坐标轴计算方式应该为Left\+(Width/2\)。
* 实验中计算出的X轴坐标与手动拖动滑块的真实坐标可能会有一个误差，但若干次测试后观察到这个误差波动不大，在一个比较固定的范围内。因此计算出的坐标值需要调整，调整值是多少？用相同的验证码图片分别在真实环境和模型中测试，得到一组数据计算出平均值，本例只取了10来组数据。
* 模型会返回一对坐标数据，本案例中只有X轴坐标的位置会用于验证，因为验证码的背景图和滑块图的高度是完全一样的，鼠标拖过中滑块只会横向移动。


核心代码：




```
    [ApiController]
    [Route("api/yolo")]
    public class YoloController : ControllerBase
    {

        private Yolo yolo;

        public YoloController()
        {
            yolo = new Yolo(new YoloOptions
            {
                OnnxModel = $"e:/4-code-space/82.yolo/models/slider.onnx",
                ModelType = ModelType.ObjectDetection,
                Cuda = false,
                GpuId = 0,
                PrimeGpu = false,
            });
        }

        [HttpPost]
        public IActionResult Detect([FromForm] IFormFile image)
        {
            try
            {
                using var memoryStream = new MemoryStream();
                image.CopyTo(memoryStream);
                var imageBytes = memoryStream.ToArray();
                using var imageStream = new MemoryStream(imageBytes);
                using var skBitmap = SKBitmap.Decode(imageStream);
                using var skImage = SKImage.FromBitmap(skBitmap);


                var results = yolo.RunObjectDetection(skImage, confidence: 0.25, iou: 0.7);
                int x = results[0].BoundingBox.Left + (results[0].BoundingBox.Width / 2);
                x-=20;//同图片相比wq yolov5的结果对比，平均需要减20
                Console.WriteLine($"识别结果：{x}");
                return Ok(x);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"识别错误：{ex.Message}");
                return BadRequest(ex.Message);
            }
        }
    }
```


3\. 将http请求路由到控制器，修改Program.cs，在app.Run()前面加一句app.MapControllers()。本项目sdk版本为8\.0\.403。


 三、验证模型计算结果


1\. 根据第二步计算出的结果模拟生成验证数据，按照接口要求的数据格式拼接http报文发往服务端验证识别结果是否正确。打开chrome开发者工具观察真实验证数据，多次实验踩坑后总结验证数据有以下特征：


* 滑块验证码的验证是在服务端进行的，鼠标拖动滑块过程中会产生一系列轨迹数据，这个轨迹数据记录是js实现的，验证过程中将验证码id和轨迹数据一并发往服务端，服务端验证通过返回目标数据。
* 轨迹数据有4个属性x、y、t、type,分别表示横坐标、纵坐标、时间、鼠标动作(down、move、up)。
* y坐标值不会参与验证，但轨迹数据中y值不是恒定不变的。因为横向拖动鼠标形成的不是绝对的横线而是近似横线，最终的y值是围绕0上下波动的，但波动范围不能太大。
* 轨迹数据的t值的间距由小变大，再由大变小。因为拖动滑块是由慢到快，再由快到慢最终停止的过程。最后一个轨迹点的x值只需要接近真实值即可。
* 轨迹数据的x值由0逐渐增大到接近实际值(模型返回的缺口位置)，接近后还可以再小距离反向远离实际值。因为真实环境中滑块可能拖动过度再回调。


    按上以分析生成模拟数据，java实现代码如下:




```
private List generateTrajectory(Integer width) {
    List trajectories = new ArrayList<>();
    Random random = new Random();
    int x = 0, y = 0, tmp = 0, t = 0;
    t = random.nextInt(50) + 50;
    // 按下鼠标
    trajectories.add(new MouseTrajectory(x, y, "down", t));
    // 拖动鼠标
    while (x < width) {
        x += random.nextInt(5) + 2;
        tmp = random.nextInt(100) % 4;
        if (tmp == 0) {
            y = random.nextInt(7) - 3;
        }
        t += random.nextInt(60) + 20;
        if (x > width) {
            x = width;
        }
        trajectories.add(new MouseTrajectory(x, y, "move", t));
    }
    // 释放鼠标
    trajectories.add(new MouseTrajectory(x, y, "up", t));
    return trajectories;
}
```


2\. 将生成的轨迹数据和其它查询参数按照目标接口格式组装后发往服务端。http报文的header值保持跟chrome开发者工具中的数据完全一致，尽量模拟真实环境。最终得到正确的服务端响应，这个案例中的返回数据是json格式字符串的base64编码值，需要再解码。


为了不暴露案例中的系统接口地址，只截图小部分返回数据和模拟生成的轨迹数据：


![](https://img2024.cnblogs.com/blog/574644/202411/574644-20241116094537745-509341207.png)


总结实验成功的两个关键点：


1\. 模型识别准确。训练模型用了真实环境验证码图片约300张。


2\. 轨迹验证数据尽量接近真实数据。


 


