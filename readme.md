## 登录相关

使用统一身份认证登录，登录信息位于`authorization/config.json`，共两个字段

- `username`为用户名的 base64 加密结果
- `password`为用户密码的 base64 加密结果

## TD 查询

直接运行`tdQuery.py`即可，结果形如:

```json
[
    {
        "学期": "2021-2022 春季学期",
        "TD次数": "48"
    },
    {
        "学期": "2021-2022 秋季学期",
        "TD次数": "48"
    }
]
```

## 流量查询

直接运行`trafficQuery.py`即可，结果形如:

```json
{
    "免费流量": {
        "总流量": "30 G",
        "已用": "0 G"
    },
    "赠送流量": {
        "总流量": "0 G",
        "可用": "0 G"
    }
}
```

## 自动选课

实例化`Selector`对象完成选课:

- 传入参数的说明见`utils/courseSelector.py`
- 调用`select`方法发出一次选课请求

多线程选课（比较暴力，请酌情使用）:

- 可参考`courseSelect.py`中的写法
- `THREADS`和`GAP`分别表示每一轮请求的线程数和两轮请求之间的时间

## 自动博雅

提供了以下函数:

- `get_user_profile`: 查看个人信息
- `query_news_list`: 查看通知公告
- `fore_course_query`: 查看课程预告（还不可选课）
- `selectable_course_query`: 查看当前可选择的课程
- `unselectable_course_query`: 查看当前可退选的课程
- `current_chosen_course_query`: 查看本学期已选的课程
- `history_chosen_course_query`: 查看本学期之前已选的课程
- `check_frontend_sign`: 检测选课平台前端 MD5 是否发生变化，防止官方修改 API 钓鱼
- `unselect`: 提供一个 id，立即退选 id 对应的课程直到成功或被手动终止，适用于当前可退选的课程
- `select`: 提供一个 id，立即选择 id 对应的课程直到成功或被手动终止，适用于当前可选择的课程
- `super_select`: 提供一个 id，智能选择 id 对应的课程，可自动停止，适用于预告的课程

随机请求秘钥:

- 在实例化`Selector`对象时通过`use_random_key`启动随机请求秘钥
- 默认不启动随机请求秘钥，以此省去 AES 轮秘钥生成的时间

旧版查询接口:

- 目前博雅系统通过`queryStudentSemesterCourseByPage`接口统一获取课程信息，但此前用于获取课程预告和可选课程的`queryForeCourse`和`querySelectableCourse`接口仍然存在
- `fore_course_query`、`selectable_course_query`和`unselectable_course_query`函数基于新接口获取课程预告、可选课程和可退选课程
- `Selector`类提供了`fore_course_query_old`和`selectable_course_query_old`方法通过旧接口获取获取课程预告和可选课程
- 旧接口不稳定，有概率返回空列表，`fore_course_query_old`和`selectable_course_query_old`方法通过多次请求取最长的方式解决此问题

建议时间模型:

- 对于一个尚未到选课时间的课程，定义以下三个建议时间

    - 登录时间: 在此时刷新登录状态，保证选课过程中登录令牌不会失效
    - 开始时间: 在此时开始发送选课请求，直到成功选课或到达终止时间
    - 终止时间: 若在此时程序没有结束，认为选课失败，终止选课并退出

- 本选课脚本对上述三个时间的计算方式如下，`Selector`类提供了`suggest_time`用于获取上述三个时间

    - 登录时间: 选课开始时间向前75秒
    - 开始时间: 选课开始时间向前30秒
    - 终止时间: 选课开始时间向后270秒

智能选课策略:

- 更新阶段

    - 刷新建议时间，以防止选课时间被暗改
    - 若建议登录时间距当前超过 12 小时，等待 180 分钟并进入更新阶段
    - 若建议登录时间距当前超过 6 小时，等待 60 分钟并进入更新阶段
    - 若建议登录时间距当前超过 2 小时，等待 30 分钟并进入更新阶段
    - 若上述条件不满足，进入冻结阶段

- 冻结阶段

    - 以当前的建议时间为最终建议时间，此后不再更新
    - 直接进入等待阶段

- 等待阶段

    - 若未到建议登录时间，等待 1 秒并进入等待阶段
    - 若上述条件不满足，产生一个`Selector`对象，进入选课阶段

- 选课阶段

    - 若未到建议开始时间，等待 1 秒并进入选课阶段
    - 若处于建议开始时间和建议终止时间之间，发出一次选课请求
        - 若选课成功，直接退出
        - 若选课失败，等待 1 秒，进入选课阶段
    - 若上述条件不满足，程序退出
