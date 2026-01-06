### src/controllers/factory.py作为应用中所有模型推理组件的集中工厂/单例容器
```
class ModelFactory(object):
    m_archive_cls = None
    m_moire_cls = None

    @classmethod
    def load(cls):
        cls.exector = ThreadPoolExecutor(max_workers=config.common_model_thread_pool_num)
        cls.load_archive_cls()
        cls.load_moire_cls()

    @classmethod
    #模型加载：懒加载，单例
    def load_moire_cls(cls):
        if cls.m_moire_cls is not None:
            return
        from algorithm.moire_cls.detect import MoireClsInfer
        cls.m_moire_cls = MoireClsInfer()
```
每个类变量（如m_moire_cls对应一个模型用于保存该模型的推理实例），服务端中所有要调用模型处理的时候都从这个ModelFactory获取推理实例

Triton 推理服务Triton Inference Server

### 2.服务端中调用模型流程
在服务端中，一般调用ModelFactory对应模型的推理实例的detect方法，如
```
def _run_moire_cls(self, ctx, img):
        return ModelFactory.m_moire_cls.detect(ctx, img)
```
```
@algorithm_detect('moire_cls')
    def detect(self, ctx, image):
        img = self.preprocess(image)
        outs = mm.infer_engine.moire_cls_model_infer(ctx, img)
        result = np.argmax(outs, 1)
        if sum(result) < 3:
            return True  # 有摩尔纹
        return False
```
在mm.infer_engine.moire_cls_model_infer方法内部真正把请求发送到 Triton 推理服务并返回模型输出（通常为 numpy 数组

### 3.判断扫描图像是否为文字场景
```
if self.custom_class and 'is_word' in self.custom_class:
        is_word = self.custom_class['is_word']
    else:
        is_word = self._run_word_cls(ctx, img_bgr)
        if not is_word:
            points = self._run_quadrilateral(ctx, img_bgr)
            if points is not None:
                # tmp_img_bgr = self._run_persperctive(img_bgr, points)
                tmp_img_bgr = self._run_cal_height_width(ctx, img_bgr, points)
                tmp_img = cv2.cvtColor(tmp_img_bgr, cv2.COLOR_BGR2RGB)
                is_word = self._run_word_cls(ctx, tmp_img_bgr) # bgr
                if self.use_quad:
                    img_bgr = tmp_img_bgr
                    img = tmp_img
            quadrilateral_flag = True
    class_result['is_word'] = is_word
```
值得注意的是，调用了按检测到的四边形将图像裁切模型进行图像四边形裁切，这样做除背景噪声：把干扰物（桌面、手、阴影）移出，减少误判。
放大文本区域：原图中文本可能很小，裁切后文本占比变大，特征更明显。就有可能使之前不识别为文字场景，裁剪后识别为

### 
即便确认是“文字场景”，仍需要多场景分类来决定应该对这张含文字的图片执行哪些后续处理
```
if is_word: 
    # 多场景分类
    if self.use_cls_multi:
        # 如果自定义分类结果包含多场景分类结果，则使用传入的结果
        if self.custom_class and 'parent_class' in self.custom_class:
            parent_type = self.custom_class['parent_class']
            children_type = self.custom_class.get('children_class', '') if parent_type == 'card' else ''
            scene_type = ModelFactory.m_multiscene_cls.get_sence_type(parent_type, children_type)
        else:
            scene_type, parent_type, children_type = self._run_multiscene_cls(ctx, img_bgr, angle, is_recog_card)
        class_result['scene_type'] = scene_type
        class_result['parent_class'] = parent_type
        class_result['children_class'] = children_type
```
