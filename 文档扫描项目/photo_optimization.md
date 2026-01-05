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
