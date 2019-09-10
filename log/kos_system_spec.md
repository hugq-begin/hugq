# 0 概览
KOS作为KPU（(我们自己实现的硬件)）的运行系统，实现很多kernel(如，ts_sum, ts_add, ts_sub)等等，为了调用这些kernel，将需要处理数据以及参数传给硬件，同时读取运算结果，KOS提供一个类给上层应用，该类叫KOS类。

# 1 KOS
引入`kos`的头文件`api/cxx_api.h`

-|+
:--:|:--:
data_master_:shared_ptr<DataMaster>|add_node(...):Node*
graph_master_:shared_ptr<GraphMaster>|add_data(...)