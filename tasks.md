执行待办事项列表
1. 执行策略
实现方法: 增量式、垂直切片开发。每个里程碑产出可独立验证的功能模块，而非水平层（如先完成所有数据库表再写UI）。

依赖逻辑:

认证是基础（所有页面需要登录）

Supabase配置优先（数据库表、RLS、Storage）

地图CRUD是核心价值，优先于社交功能

AI功能可并行开发（API Routes独立）

分阶段: 5个里程碑

2. 里程碑
里程碑	名称	目标	完成标准
M1	基础框架与认证	项目可运行，用户能注册登录	注册/登录流程完整，受保护页面跳转正常
M2	地图与点位CRUD	用户能搜索店铺、添加评价、在地图上看到点位	完整的“搜索→填表→保存→地图展示”流程
M3	AI功能集成	AI标签推荐和语义搜索可用	标签推荐返回3-5个标签，语义搜索能匹配情绪
M4	社交与协作	好友系统和邀请协助可用	好友添加、邀请填写、共同回忆展示完整流程
M5	个人中心与上线	月度总结、打卡记录列表完成，部署到生产	所有验收标准通过，Vercel生产环境可用
3. 按里程碑划分的任务分解
里程碑 1: 基础框架与认证
目标

搭建Next.js项目，配置Tailwind CSS和shadcn/ui

配置Supabase项目，完成数据库表结构

实现邮箱注册/登录功能

配置路由保护中间件

任务 1.1: 项目初始化
目的

创建Next.js + TypeScript + Tailwind CSS项目

配置shadcn/ui组件库

范围

初始化项目

配置Tailwind CSS

安装并配置shadcn/ui

配置路径别名（@/*）

建议的实现说明

使用create-next-app，选择TypeScript和Tailwind CSS

按照shadcn/ui文档初始化（npx shadcn-ui@latest init）

添加基础布局组件（Layout.tsx）

验收标准

npm run dev成功启动，访问http://localhost:3000显示默认页面

Tailwind CSS生效（修改背景色可见变化）

shadcn/ui Button组件可正常导入和使用

建议的提交粒度

提交1: Next.js + TypeScript + Tailwind初始化

提交2: shadcn/ui配置 + 基础布局组件

依赖关系

无

风险 / 失败模式

Node版本不兼容 → 使用Node 18.x或更高版本

任务 1.2: Supabase项目配置
目的

创建Supabase项目

执行数据库迁移，创建所有表

配置RLS策略

创建Storage桶

范围

在Supabase Dashboard创建新项目

执行以下SQL脚本：

profiles表

reviews表

collaborations表

friendships表

配置RLS策略

创建review-imagesStorage桶，设置公开读权限

建议的实现说明

使用Supabase SQL编辑器执行迁移

保存所有SQL脚本到supabase/migrations/目录

配置Storage桶的bucket权限：公开读，认证用户写

验收标准

所有表在Supabase Database中可见

RLS策略已启用

Storage桶review-images存在，可公开读取

可通过Supabase Client查询profiles表（空结果）

建议的提交粒度

提交1: 添加迁移SQL文件到supabase/migrations/

提交2: 更新README包含Supabase配置说明

依赖关系

无（独立于代码）

风险 / 失败模式

表名/字段名与代码不一致 → 使用TypeScript类型同步验证

RLS策略配置错误导致无法查询 → 先禁用RLS测试，再逐条启用

任务 1.3: Supabase客户端配置
目的

配置Supabase浏览器客户端和服务器客户端

创建环境变量文件

范围

安装@supabase/supabase-js

创建lib/supabase/client.ts（浏览器客户端）

创建lib/supabase/server.ts（服务器客户端，用于middleware）

创建.env.local模板

建议的实现说明

浏览器客户端用于客户端组件中的CRUD操作

服务器客户端用于middleware和API Routes

环境变量：NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY

验收标准

客户端初始化无报错

可调用supabase.auth.getSession()获取session

环境变量正确加载

建议的提交粒度

提交1: 安装依赖 + 创建客户端文件

提交2: 添加环境变量模板.env.example

依赖关系

任务1.2（Supabase项目需已创建）

风险 / 失败模式

环境变量未配置导致运行时错误 → 添加启动前检查脚本

任务 1.4: 注册/登录页面实现
目的

实现邮箱注册和登录页面

集成Supabase Auth

范围

创建app/(auth)/login/page.tsx

创建app/(auth)/register/page.tsx

使用React Hook Form + Zod做表单验证

注册成功后自动创建profiles记录（使用触发器或代码）

建议的实现说明

使用shadcn/ui的Card、Input、Button组件

注册流程：调用supabase.auth.signUp()，验证邮箱后自动登录

登录流程：调用supabase.auth.signInWithPassword()

使用Supabase触发器自动创建profiles：当auth.users插入时，自动插入profiles

sql
-- 触发器SQL
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, nickname)
  VALUES (NEW.id, split_part(NEW.email, '@', 1));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
验收标准

注册页面表单验证生效（邮箱格式、密码长度≥6）

注册成功后自动登录，跳转到地图页

登录成功后跳转到地图页

profiles表自动创建对应记录

建议的提交粒度

提交1: 注册页面 + 表单验证

提交2: 登录页面 + 触发器SQL

提交3: 添加登录/注册成功后的跳转逻辑

依赖关系

任务1.1（项目初始化）

任务1.3（Supabase客户端）

风险 / 失败模式

邮箱验证邮件可能进入垃圾箱 → 添加提示文案

触发器SQL执行失败 → 在注册成功后手动创建profiles作为fallback

任务 1.5: 路由保护中间件
目的

实现middleware，保护需要登录的页面

未登录用户重定向到登录页

范围

创建middleware.ts

配置受保护路由（/dashboard下的所有页面）

公开路由：/login, /register

建议的实现说明

使用Supabase服务器客户端验证session

中间件在Edge Runtime运行

登录成功后重定向到原始请求页面（使用redirectTo参数）

验收标准

未登录时访问/自动重定向到/login

登录后访问/正常显示地图页

登录后访问/profile正常显示

建议的提交粒度

提交1: 基础middleware实现

提交2: 添加重定向回原始页面逻辑

依赖关系

任务1.3（Supabase服务器客户端）

风险 / 失败模式

中间件在Edge Runtime下Supabase客户端配置不同 → 使用@supabase/ssr包

里程碑 2: 地图与点位CRUD
目标

集成高德地图，显示地图视图

实现高德POI搜索，用户可选择店铺

实现点位添加表单（含图片上传）

实现点位在地图上的展示

任务 2.1: 高德地图集成
目的

在Next.js中集成高德地图JS API v2.0

实现基础地图显示

范围

在index.html或Next.js Script组件中加载高德地图SDK

创建components/map/MapContainer.tsx（客户端组件）

使用动态导入禁用SSR

配置高德API Key（NEXT_PUBLIC_AMAP_KEY）

建议的实现说明

使用next/dynamic导入地图组件，ssr: false

在useEffect中初始化地图，避免hydration错误

高德地图加载回调：window.initAMap

验收标准

地图页面加载后显示高德地图

地图可缩放、拖拽

浏览器请求地理位置权限（或显示默认区域）

建议的提交粒度

提交1: 添加高德SDK加载逻辑

提交2: 创建MapContainer组件，动态导入

提交3: 地图初始化成功，显示默认视图

依赖关系

任务1.1（项目初始化）

风险 / 失败模式

Next.js SSR与高德不兼容 → 使用动态导入ssr: false

高德API Key未配置 → 添加环境变量检查

任务 2.2: 高德POI搜索组件
目的

实现店铺名称搜索（传统搜索）

返回店铺列表供用户选择

范围

创建components/search/PlaceSearch.tsx

使用高德AMap.PlaceSearch API

支持城市筛选（选填）

搜索结果列表展示（名称、地址）

建议的实现说明

使用Ant Design或shadcn/ui的Input、Button、List组件

搜索结果缓存5分钟（Map结构）

选择店铺后通过回调传递店铺数据给父组件

验收标准

输入关键词后可搜索到店铺列表

支持城市筛选（默认全国）

点击店铺后触发回调，传递店铺详情

重复搜索相同关键词时使用缓存

建议的提交粒度

提交1: 基础搜索组件 + 高德PlaceSearch调用

提交2: 添加城市筛选

提交3: 添加搜索缓存逻辑

依赖关系

任务2.1（高德地图SDK已加载）

风险 / 失败模式

高德API日调用量限制 → 前端缓存减少请求

任务 2.3: 点位添加表单
目的

实现点位添加表单

自动填充高德返回的店铺信息

包含喜爱值、情绪emoji、文字评价、图片上传

范围

创建components/review/ReviewForm.tsx

使用React Hook Form + Zod验证

集成PlaceSearch组件选择店铺

表单字段：

店铺信息（只读，来自搜索）

喜爱值（1-10，滑块或数字输入）

情绪emoji（4个按钮：🥰😌🤔😤）

文字评价（可选，textarea）

图片上传（最多3张）

建议的实现说明

使用shadcn/ui的Slider、RadioGroup、Textarea、Upload组件

表单验证：店名必填，喜爱值1-10

店铺选择后自动填充表单

验收标准

店铺选择后，店名、地址、电话等字段自动填充

喜爱值滑块可调节1-10

情绪emoji可单选

图片可上传（前端预览，暂不保存到Storage）

建议的提交粒度

提交1: 表单基础结构 + React Hook Form

提交2: 集成PlaceSearch，自动填充

提交3: 添加喜爱值滑块和情绪emoji

提交4: 添加图片上传（预览功能）

依赖关系

任务2.2（PlaceSearch组件）

风险 / 失败模式

表单字段过多用户放弃 → 简化布局，折叠高级选项

任务 2.4: AI标签推荐API Route
目的

创建Next.js API Route调用通义千问API

生成3-5个推荐标签

范围

创建app/api/ai/tags/route.ts

使用Edge Runtime

构造Prompt，调用千问API（qwen-turbo）

返回{ tags: string[] }

建议的实现说明

Prompt模板："基于以下餐饮店信息，生成3-5个简短的标签（每个标签2-5个字，用#开头）：店名：{shopName}，用户评价：{textReview}。只返回标签，用逗号分隔。"

解析千问返回结果，提取标签

降级：API失败时返回默认标签["#值得一试", "#还不错"]

验收标准

POST /api/ai/tags返回正确的标签数组

请求超时5秒时返回降级标签

千问API Key存储在环境变量，不暴露

建议的提交粒度

提交1: API Route骨架 + 环境变量读取

提交2: 千问API调用 + 响应解析

提交3: 添加超时降级逻辑

依赖关系

无（独立API）

风险 / 失败模式

千问API Key未配置 → 添加检查，返回友好错误

千问返回格式不稳定 → 使用正则提取#开头的词

任务 2.5: 图片上传到Supabase Storage
目的

实现图片上传到Supabase Storage

返回公开URL

范围

在ReviewForm中集成Supabase Storage上传

上传路径：{userId}/{timestamp}/{filename}

上传成功后保存URL到表单的images数组

建议的实现说明

使用supabase.storage.from('review-images').upload()

生成唯一文件名：Date.now() + '-' + file.name

上传前检查文件大小（≤2MB）和类型（image/jpeg, image/png）

使用next/image显示缩略图

验收标准

图片选择后自动上传到Supabase Storage

上传成功后显示缩略图

上传失败时显示错误提示

超过3张时禁用上传按钮

建议的提交粒度

提交1: Storage上传函数

提交2: 集成到表单，上传进度显示

提交3: 添加文件大小和类型验证

依赖关系

任务1.2（Supabase Storage桶已创建）

任务2.3（表单已有图片上传区域）

风险 / 失败模式

大文件上传慢 → 限制2MB，添加loading状态

任务 2.6: 点位保存与列表查询
目的

实现reviews表的插入和查询

使用TanStack Query管理服务端状态

范围

创建hooks/useReviews.ts，封装查询和mutation

表单提交时插入reviews表

地图页面查询当前用户的所有reviews

建议的实现说明

安装@tanstack/react-query

创建QueryClientProvider包装应用

useReviews hook包含：

useReviews（查询）

useAddReview（插入）

useUpdateReview（更新）

useDeleteReview（删除）

验收标准

表单提交后，reviews表新增记录

提交成功后，TanStack Query缓存失效，重新查询

查询返回正确的用户点位列表

建议的提交粒度

提交1: 安装React Query + 配置Provider

提交2: 创建useReviews hook

提交3: 表单集成useAddReview

提交4: 地图页面集成useReviews查询

依赖关系

任务2.3（表单）

任务2.5（图片上传，获取URL）

风险 / 失败模式

RLS策略阻止插入 → 检查用户认证和RLS配置

任务 2.7: 地图点位展示
目的

在地图上显示用户添加的点位

按喜爱值区分Marker颜色

点击Marker显示店铺详情卡片

范围

创建components/map/MapMarker.tsx

喜爱值颜色映射：

1-3分：灰色

4-7分：橙色

8-10分：红色

点击Marker弹出信息窗口

建议的实现说明

使用高德地图的AMap.Marker和AMap.InfoWindow

颜色映射通过自定义Icon实现

信息窗口显示：店名、喜爱值、情绪emoji、图片缩略图

验收标准

每个点位在地图上显示对应颜色的Marker

点击Marker弹出信息窗口

信息窗口显示完整点位信息

建议的提交粒度

提交1: Marker渲染逻辑 + 颜色映射

提交2: 信息窗口展示

提交3: 点击Marker时地图定位

依赖关系

任务2.1（地图容器）

任务2.6（reviews查询）

风险 / 失败模式

大量Marker导致地图卡顿 → 使用点聚合（后续迭代）

里程碑 3: AI功能集成
目标

AI标签推荐在表单中可用

语义搜索API完成

前端语义搜索组件可用

任务 3.1: 表单集成AI标签推荐
目的

在点位表单中添加“AI推荐标签”按钮

调用API获取标签，用户可高亮选中

范围

在ReviewForm中添加按钮和标签选择区域

调用/api/ai/tags获取标签

标签可点击高亮/取消选中

支持自定义添加标签

建议的实现说明

使用shadcn/ui的Badge组件显示标签

标签状态：未选中（灰色边框），选中（蓝色背景）

选中标签存入aiTags字段

自定义标签输入框，Enter添加

验收标准

点击按钮后3秒内返回标签列表

标签可点击高亮/取消

可手动添加自定义标签

保存时标签正确存储到aiTags字段

建议的提交粒度

提交1: 添加标签按钮和API调用

提交2: 标签高亮选中逻辑

提交3: 自定义标签添加

依赖关系

任务2.4（AI标签API）

任务2.3（表单）

风险 / 失败模式

API响应慢 → 添加loading状态，5秒超时降级

任务 3.2: 语义搜索API Route
目的

创建语义搜索API，解析自然语言意图

返回意图类型和值

范围

创建app/api/ai/semantic/route.ts

支持意图：emotion（🥰😌🤔😤）、tag（#标签）、text（全文搜索）

调用千问API解析用户输入

建议的实现说明

Prompt模板："分析用户的美食搜索意图：'{query}'。返回JSON格式：{\"intent\": \"emotion|tag|text\", \"value\": \"对应值\"}"

示例：

"惊喜的餐厅" → {"intent": "emotion", "value": "🥰"}

"#安静的地方" → {"intent": "tag", "value": "#安静"}

"适合约会的餐厅" → {"intent": "text", "value": "适合约会"}

验收标准

输入“惊喜的餐厅”返回emotion类型，value为🥰

输入“踩雷”返回emotion类型，value为😤

输入“#辣味十足”返回tag类型

输入“适合拍照”返回text类型

建议的提交粒度

提交1: API Route骨架

提交2: 千问API调用 + 意图解析

提交3: 添加错误降级（默认text搜索）

依赖关系

无

风险 / 失败模式

千问返回JSON格式不稳定 → 使用正则二次解析

任务 3.3: 前端语义搜索组件
目的

实现语义搜索输入框

调用语义搜索API，匹配本地reviews

返回匹配点位列表

范围

创建components/search/SemanticSearch.tsx

输入框（与店铺名称搜索分开）

调用/api/ai/semantic解析意图

根据意图查询本地reviews：

emotion → 匹配emotion_emoji

tag → 匹配ai_tags或custom_tags

text → 全文搜索text_review

显示匹配结果列表

建议的实现说明

使用TanStack Query缓存搜索结果

本地查询使用Supabase Client的ilike或数组包含操作

结果列表可点击，地图定位

验收标准

输入“惊喜的餐厅”返回所有🥰情绪的点位

输入“#辣味十足”返回包含该标签的点位

输入“适合拍照”返回文字评价中包含该词的点位

点击结果后地图定位到对应Marker

建议的提交粒度

提交1: 语义搜索组件 + API调用

提交2: 本地查询逻辑（意图映射）

提交3: 结果列表 + 地图定位

依赖关系

任务3.2（语义搜索API）

任务2.6（reviews查询）

风险 / 失败模式

意图解析错误导致无结果 → 降级为全文搜索

里程碑 4: 社交与协作
目标

好友系统（添加、确认、列表）

邀请协助功能

共同回忆模块

任务 4.1: 好友系统
目的

实现通过邮箱添加好友

好友申请/确认流程

范围

创建components/social/FriendList.tsx

搜索用户（通过邮箱）

发送好友申请（插入friendships表，status='pending'）

待处理申请列表

确认/拒绝申请（更新status）

好友列表显示

建议的实现说明

需要查询profiles表获取用户昵称

使用Supabase Realtime订阅friendships表变更

好友列表显示头像（首字母）和昵称

验收标准

可通过邮箱搜索到用户

发送申请后对方收到通知（页面内）

对方确认后，双方好友列表更新

好友列表显示已确认的好友

建议的提交粒度

提交1: friendships表CRUD操作

提交2: 搜索用户组件

提交3: 好友列表 + 申请/确认UI

提交4: Realtime订阅实时更新

依赖关系

任务1.2（friendships表已存在）

风险 / 失败模式

邮箱搜索可能返回多个用户 → 显示列表让用户选择

任务 4.2: 邀请协助功能
目的

用户在添加点位时可邀请好友共同评价

好友收到通知并填写评价

双方评价关联

范围

在ReviewForm中添加“邀请好友”开关

选择好友（从好友列表）

创建collaborations记录

好友收到Realtime通知

好友填写评价后关联collaborationId

双方地图显示“共同去过”标记

建议的实现说明

通知使用Supabase Realtime订阅collaborations表

好友评价提交时，检查collaborationId是否匹配

共同去过标记：双人头像图标

验收标准

添加点位时可选择邀请好友

被邀请好友收到通知（页面内提示）

好友填写评价后，双方reviews记录关联相同collaborationId

地图上该店铺显示双人头像图标

建议的提交粒度

提交1: ReviewForm添加邀请好友UI

提交2: collaborations表操作

提交3: Realtime通知订阅

提交4: 地图共同去过标记

依赖关系

任务4.1（好友列表）

任务2.3（表单）

风险 / 失败模式

Realtime连接不稳定 → 添加轮询降级

任务 4.3: 店铺详情卡片与共同去过Tab
目的

点击地图Marker显示店铺详情卡片

“共同去过”Tab显示邀请协助完成的评价对

范围

创建components/map/ShopDetailCard.tsx

显示店铺基础信息（高德数据）

“共同去过”Tab：查询相同collaborationId的Review对

无共同回忆时不显示Tab

建议的实现说明

使用shadcn/ui的Card和Tabs组件

共同回忆查询：根据poiId查询reviews，筛选有collaborationId且双方都完成的

显示双方头像、昵称、喜爱值、情绪、评价

验收标准

点击Marker弹出卡片，显示店铺信息

有共同回忆时显示“共同去过”Tab

Tab内显示双方评价

无共同回忆时Tab隐藏

建议的提交粒度

提交1: ShopDetailCard组件

提交2: 共同去过查询逻辑

提交3: 集成到地图点击事件

依赖关系

任务2.7（地图Marker）

任务4.2（collaborations）

风险 / 失败模式

查询性能问题 → 添加索引

任务 4.4: 共同回忆模块
目的

个人中心展示与好友的共同回忆

选择好友后展示一起去过的店铺列表

生成总结和音乐推荐

范围

创建components/profile/MemoryList.tsx

好友列表（有共同探店记录的）

选择好友后查询共同店铺

口味相似度计算（Jaccard）

情绪重合度计算

调用音乐推荐API

建议的实现说明

口味相似度：|A ∩ B| / |A ∪ B|，A和B为标签数组

情绪重合度：相同情绪标签的比例

音乐推荐调用/api/ai/music，type='memory'

验收标准

显示有共同探店记录的好友列表

选择好友后展示共同店铺列表

显示口味相似度和情绪重合度

显示音乐推荐，支持“换一首”

建议的提交粒度

提交1: 好友列表（有共同记录）

提交2: 共同店铺查询

提交3: 相似度计算

提交4: 音乐推荐集成

依赖关系

任务4.2（collaborations）

任务3.4（音乐推荐API，M3中已实现）

风险 / 失败模式

共同店铺过多导致页面慢 → 分页加载

里程碑 5: 个人中心与上线
目标

打卡记录列表

月度总结模块

部署到Vercel生产环境

完成所有验收测试

任务 5.1: 打卡记录列表
目的

个人中心展示所有点位记录

只读展示，不支持点击跳转

范围

创建components/profile/ReviewList.tsx

查询当前用户所有reviews

每项显示：情绪emoji、喜爱值、图片缩略图、店铺名、评价摘要

分页加载（每页20条）

建议的实现说明

使用TanStack Query的无限查询

图片使用next/image，添加懒加载

按created_at倒序排列

验收标准

列表显示所有点位

每项显示正确信息

滚动到底部自动加载下一页

图片懒加载生效

建议的提交粒度

提交1: ReviewList组件 + 查询

提交2: 分页/无限滚动

提交3: 图片懒加载优化

依赖关系

任务2.6（reviews查询）

风险 / 失败模式

图片过多影响性能 → 使用next/image优化

任务 5.2: 月度总结模块
目的

按月统计用户探店数据

展示统计卡片和音乐推荐

范围

创建components/profile/MonthlySummary.tsx

月份选择器（切换月份）

统计：新增点位数量、平均喜爱值、最常标签Top3、情绪分布

调用音乐推荐API（type='monthly'）

展示当月打卡记录列表（店名、日期、喜爱值、情绪）

建议的实现说明

统计查询使用Supabase聚合函数（count, avg, 分组）

情绪分布：按emotion_emoji分组计数

最常标签：使用unnest展开标签数组统计

验收标准

默认显示当前月份

切换月份后数据更新

统计卡片数据正确

音乐推荐可“换一首”

建议的提交粒度

提交1: 月度总结统计查询

提交2: 统计卡片UI

提交3: 音乐推荐集成

提交4: 月份切换器

依赖关系

任务2.6（reviews查询）

任务3.4（音乐推荐API）

风险 / 失败模式

聚合查询性能慢 → 添加created_at索引

任务 5.3: 生产环境部署到Vercel
目的

将项目部署到Vercel生产环境

配置所有环境变量

验证生产环境功能

范围

连接Git仓库到Vercel

配置环境变量：

NEXT_PUBLIC_SUPABASE_URL

NEXT_PUBLIC_SUPABASE_ANON_KEY

NEXT_PUBLIC_AMAP_KEY

DASHSCOPE_API_KEY

配置高德API Key域名白名单（生产域名）

验证所有功能

建议的实现说明

使用Vercel CLI或Git集成部署

配置自动部署（main分支）

添加vercel.json配置（可选）

验收标准

部署成功，访问生产域名正常

注册/登录功能正常

地图加载正常

AI功能正常

好友和邀请功能正常

建议的提交粒度

提交1: 添加vercel.json配置

提交2: 更新README部署说明

依赖关系

所有前置任务

风险 / 失败模式

环境变量未配置 → 使用Vercel Dashboard添加

高德API Key域名白名单未配置 → 添加生产域名

任务 5.4: MVP完成验收测试
目的

执行product-brief.md中的所有验收标准

修复发现的Bug

范围

按照验收标准逐项测试

记录Bug并修复

确保所有标准通过

建议的实现说明

创建测试检查清单

测试环境：生产环境

测试用户：至少2个账号测试社交功能

验收标准

product-brief.md中所有验收标准通过

无阻塞性Bug

核心流程（注册→添加→邀请→月度总结）可完成

建议的提交粒度

提交1: 修复测试中发现的Bug

提交2: 最终验收通过确认

依赖关系

任务5.3（生产环境）

风险 / 失败模式

发现重大问题 → 评估是否阻塞MVP发布

4. 横向检查
在整个执行过程中，每个任务完成后应验证以下内容：

类型安全
无any类型（除非必要，加注释说明）

Supabase查询结果有正确类型

代码质量
ESLint无错误

Prettier格式化通过

无console.log残留（生产环境）

加载/错误/空状态
所有异步操作有loading状态

所有API调用有错误处理

列表/搜索无结果时显示空状态文案

身份验证
受保护页面有middleware拦截

未登录时API返回401（通过RLS）

响应式设计
移动端布局（375px宽度）可用

表单在小屏幕下正常显示

环境变量
所有环境变量在.env.example中有示例

生产环境变量已配置

5. MVP完成的定义
功能完整性
用户可注册/登录

用户可搜索店铺并添加评价（含图片、喜爱值、情绪、标签）

AI标签推荐可用（3-5个标签）

地图显示用户点位，按喜爱值分色

语义搜索可用（支持情绪、标签、全文）

店铺名称搜索可用（高德POI）

用户可添加/删除/编辑自己的评价

用户可通过邮箱添加好友

用户可邀请好友共同评价店铺

被邀请好友可填写评价，形成共同回忆

店铺详情卡片显示共同去过评价

个人中心显示打卡记录列表

个人中心显示共同回忆模块（好友分组 + 音乐推荐）

个人中心显示月度总结（统计 + 音乐推荐）

质量基线
无阻塞性Bug

核心流程在主流浏览器（Chrome、Firefox、Safari）可用

所有API调用有降级方案（AI超时、地图限额）

部署就绪
项目部署到Vercel生产环境

自定义域名配置（可选）

环境变量正确配置

高德API域名白名单包含生产域名

6. 推荐的AI编码工作模式
执行原则
一次一个任务：不要同时实现多个任务

按顺序执行：严格按里程碑顺序，不要跳过依赖