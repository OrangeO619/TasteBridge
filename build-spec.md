构建规范
1. 技术摘要
MVP定位: 基于Web的探店记忆共享平台，核心能力为地图点位CRUD、AI辅助标签、语义搜索、好友协作

主要技术目标:

前端: Next.js 14 + React 18 + TypeScript + Tailwind CSS，App Router架构

后端: Supabase BaaS（零运维，内置认证/数据库/存储/实时）+ Next.js API Routes（AI代理）

地图: 高德地图JS API v2.0

部署: Vercel（统一平台，自动CI/CD）

关键实现假设:

Supabase免费额度足够MVP（100-200活跃用户）

高德地图JS API日调用5000次足够（Key暴露前端可接受）

千问API Key严禁暴露前端，必须通过Next.js API Route代理

图片不压缩，单张≤2MB，用户上传≤3张/点位

Next.js App Router支持客户端组件和服务端组件分离

2. 推荐技术栈
层级	技术选型	理由
前端框架	Next.js 14 (App Router) + React 18 + TypeScript	SSR/SSG可选，API Routes内置，Vercel原生支持
样式方案	Tailwind CSS + class-variance-authority	原子化CSS，快速开发，与Next.js完美集成
UI组件	shadcn/ui (基于Radix UI)	无头组件，完全可控，Tailwind友好
地图服务	高德地图JS API v2.0 + @react-amap/amap	React封装，POI搜索和逆地理编码完善
BaaS	Supabase	托管PostgreSQL + 认证 + 存储 + 实时，零后端代码
AI代理	Next.js API Routes (App Router)	与前端同一项目，无需额外Vercel Function
状态管理	Zustand + React Query (TanStack Query)	轻量级，服务端状态+客户端状态分离
数据请求	@supabase/supabase-js + TanStack Query	缓存+乐观更新+实时订阅
表单处理	React Hook Form + Zod	类型安全的表单验证
图片上传	Supabase Storage + next/image	直接上传，返回公开URL，Next.js优化
部署	Vercel	自动CI/CD，预览环境，免费额度，Next.js原生
监控	Vercel Analytics + Sentry	性能监控 + 错误追踪
关键安全说明
Key类型	存储位置	暴露风险	防护措施
高德API Key	前端环境变量（NEXT_PUBLIC_AMAP_KEY）	低（域名白名单限制）	配置HTTP Referer白名单
千问API Key	服务器环境变量（.env.local）	高（无限制调用）	严禁暴露在前端，仅API Routes使用
Supabase anon Key	前端环境变量（NEXT_PUBLIC_SUPABASE_ANON_KEY）	低（RLS策略保护）	依赖RLS行级安全
3. 系统范围
需要构建的系统/模块
Next.js前端应用: App Router架构，包含地图、表单、个人中心等页面

Next.js API Routes: 3个API端点（标签推荐/语义解析/音乐推荐）

Supabase配置: 数据库表、RLS策略、Storage桶、Realtime订阅

有意排除的系统/模块
独立后端服务: 所有CRUD使用Supabase Client直连

消息队列/任务调度: MVP不需要，好友通知用Realtime

CDN/图片处理: 使用Supabase Storage原生图片转换API（?width=100）

日志聚合系统: 使用Vercel自带日志

单元测试框架: MVP阶段手动测试 + 集成测试

SSR复杂逻辑: MVP以客户端组件为主，SSR仅用于简单页面

4. 高层架构
text
┌─────────────────────────────────────────────────────────────────┐
│                         Vercel 部署平台                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Next.js 14 (App Router)                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │  │
│  │  │ 客户端组件   │  │ 服务端组件   │  │  API Routes      │  │  │
│  │  │ (use client)│  │ (默认SSR)   │  │  /api/ai/*       │  │  │
│  │  └──────┬──────┘  └─────────────┘  └────────┬─────────┘  │  │
│  │         │                                    │             │  │
│  │         │ 1. 地图/表单/个人中心               │ 3. AI代理    │  │
│  │         │ 2. Supabase直连                    │ (隐藏Key)    │  │
│  └─────────┼────────────────────────────────────┼─────────────┘  │
│            │                                    │                 │
│            │ HTTPS (Supabase Client)            │ 内部调用        │
│            ▼                                    ▼                 │
│     ┌─────────────┐                      ┌─────────────┐        │
│     │  Supabase   │                      │  通义千问   │        │
│     │  (BaaS)     │                      │    API     │        │
│     ├─────────────┤                      └─────────────┘        │
│     │ • Auth      │                                              │
│     │ • PostgreSQL│                                              │
│     │ • Storage   │                                              │
│     │ • Realtime  │                                              │
│     └─────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    高德地图API (前端直连)                         │
│  • POI搜索: /v3/place/text                                       │
│  • 逆地理编码: /v3/geocode/regeo                                 │
│  • Key暴露前端（域名白名单保护）                                   │
└─────────────────────────────────────────────────────────────────┘

数据流说明:
1. Next.js客户端组件直连Supabase进行CRUD（利用RLS策略）
2. 图片上传到Supabase Storage，保存URL到reviews表
3. AI请求通过Next.js API Routes代理（隐藏千问API Key）
4. 高德API前端直连（Key有域名白名单限制）
5. 好友邀请通过Supabase Realtime推送
6. 服务端组件用于SEO页面（如公开的月度总结分享页，MVP可选）
5. 核心模块
5.1 认证模块 (modules/auth)
目的: 邮箱注册/登录，会话管理

职责: 调用Supabase Auth API，管理JWT，处理验证邮件

输入: 邮箱/密码

输出: 用户session，Supabase Client实例

依赖: Supabase Client，Next.js Middleware（路由保护）

文件位置: app/(auth)/login/page.tsx, app/(auth)/register/page.tsx

5.2 地图模块 (modules/map)
目的: 地图展示、点位聚合、标记交互

职责: 初始化高德地图，渲染Marker，处理点击事件，按poiId去重

输入: 点位数据列表（reviews）

输出: 地图视图，Marker弹出卡片

依赖: 高德地图JS API，@react-amap/amap

文件位置: components/map/MapContainer.tsx, components/map/MapMarker.tsx

5.3 点位管理模块 (modules/review)
目的: 点位CRUD、图片上传、AI标签推荐

职责: 表单处理，高德POI搜索，图片上传到Supabase，调用API Routes

输入: 用户填写的评价数据

输出: 保存到reviews表，更新地图

依赖: Supabase Client，API Routes，高德POI搜索API

文件位置: components/review/ReviewForm.tsx, components/review/ReviewCard.tsx

5.4 社交模块 (modules/social)
目的: 好友系统、邀请协助、共同回忆

职责: 好友申请/确认，创建collaborations，订阅Realtime通知，聚合共同店铺

输入: 好友邮箱，邀请请求

输出: friendships表记录，collaborations表记录，实时通知

依赖: Supabase Client，Realtime订阅

文件位置: components/social/FriendList.tsx, components/social/InviteNotification.tsx

5.5 搜索模块 (modules/search)
分为两个独立子模块：

5.5.1 店铺名称搜索（传统搜索）
职责: 调用高德POI搜索API，返回实时店铺列表

输入: 店铺名称关键词（必填）+ 城市（选填）

输出: 高德API返回的店铺列表

依赖: 高德地图JS API

文件位置: components/search/PlaceSearch.tsx

5.5.2 语义搜索（AI搜索）
职责: 调用API Routes解析自然语言，匹配本地点位的标签/情绪/评价

输入: 自然语言查询

输出: 匹配的点位列表

依赖: API Routes（/api/ai/semantic）+ Supabase Client

文件位置: components/search/SemanticSearch.tsx

5.6 个人中心模块 (modules/profile)
目的: 打卡记录列表、共同回忆、月度总结

职责: 查询reviews表聚合数据，调用API Routes获取音乐推荐，展示统计

输入: 用户ID，月份筛选

输出: 统计数据，音乐推荐

依赖: Supabase Client，API Routes

文件位置: app/(dashboard)/profile/page.tsx, components/profile/

5.7 AI代理模块 (app/api/ai/*)
目的: 封装通义千问API调用，隐藏API Key

职责: 接收前端请求，构造Prompt，调用千问，返回结构化结果

输入: shopName, score, textReview, tags等

输出: JSON格式的标签/音乐推荐

依赖: 通义千问API（qwen-turbo模型）

文件位置: app/api/ai/tags/route.ts, app/api/ai/semantic/route.ts, app/api/ai/music/route.ts

5.8 中间件模块 (middleware.ts)
目的: 路由保护，认证拦截

职责: 检查用户session，重定向未登录用户

输入: Next.js请求对象

输出: 响应或重定向

依赖: Supabase Server Client

文件位置: middleware.ts

6. 数据模型
6.1 profiles表（用户资料）
sql
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  nickname TEXT DEFAULT '',
  avatar_url TEXT DEFAULT '',
  updated_at TIMESTAMP DEFAULT NOW()
);
关系: 1:1关联auth.users

索引: id（主键）

6.2 reviews表（评价记录）
sql
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  poi_id TEXT NOT NULL,
  shop_name TEXT NOT NULL,
  address TEXT NOT NULL,
  lat FLOAT8 NOT NULL,
  lng FLOAT8 NOT NULL,
  phone TEXT,
  hours TEXT,
  avg_price INT,
  amap_rating FLOAT8,
  personal_score INT NOT NULL CHECK (personal_score BETWEEN 1 AND 10),
  emotion_emoji TEXT NOT NULL CHECK (emotion_emoji IN ('🥰', '😌', '🤔', '😤')),
  images TEXT[] DEFAULT '{}',
  text_review TEXT,
  ai_tags TEXT[] DEFAULT '{}',
  custom_tags TEXT[] DEFAULT '{}',
  collaboration_id UUID,
  collaborated_with UUID REFERENCES auth.users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
索引:

user_id（查询用户点位）

poi_id（店铺聚合）

collaboration_id（关联邀请）

created_at（月度统计）

关系: user_id → auth.users，collaborated_with → auth.users

6.3 collaborations表（邀请协助）
sql
CREATE TABLE collaborations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  poi_id TEXT NOT NULL,
  initiator_id UUID NOT NULL REFERENCES auth.users(id),
  partner_id UUID NOT NULL REFERENCES auth.users(id),
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'completed')),
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);
索引:

(initiator_id, partner_id, status)（查询待处理邀请）

poi_id（店铺关联）

6.4 friendships表（好友关系）
sql
CREATE TABLE friendships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id),
  friend_id UUID NOT NULL REFERENCES auth.users(id),
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'accepted')),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, friend_id)
);
索引:

(user_id, status)（查询用户好友）

(friend_id, status)（查询好友请求）

7. API / 接口契约
7.1 高德POI搜索（前端直接调用）
typescript
// 前端直接调用高德API（Key暴露前端可接受）
const placeSearch = new AMap.PlaceSearch({
  city: city || '全国',
  pageSize: 20
});

placeSearch.search(keyword, (status, result) => {
  if (status === 'complete') {
    // result.poiList.pois 包含店铺列表
  }
});
7.2 Next.js API Routes（AI代理）
端点1: 生成标签

text
POST /api/ai/tags
Request: { 
  shopName: string, 
  category?: string,
  textReview?: string 
}
Response: { tags: string[] }
Error: 400, 500
端点2: 语义搜索解析

text
POST /api/ai/semantic
Request: { query: string }
Response: { 
  intent: 'emotion' | 'tag' | 'text',
  value: string
}
Error: 400, 500
端点3: 音乐推荐（月度/共同回忆）

text
POST /api/ai/music
Request: { 
  type: 'monthly' | 'memory',
  topTags?: string[],
  emotionCounts?: object,
  avgScore?: number,
  commonTags?: string[],
  commonEmojis?: string[],
  count?: number
}
Response: { songName: string, artist: string, reason: string }
Error: 500
7.3 Supabase Client直连
所有CRUD操作通过Supabase Client直接调用，利用RLS策略鉴权。

8. 状态和数据流
8.1 客户端状态管理
typescript
// Zustand Store (客户端状态)
interface AppStore {
  mapMode: 'my' | 'friends';
  selectedPOI: string | null;
  setMapMode: (mode: 'my' | 'friends') => void;
}

// TanStack Query (服务端状态)
const { data: reviews } = useQuery({
  queryKey: ['reviews', userId],
  queryFn: () => supabase.from('reviews').select('*').eq('user_id', userId)
});
8.2 Next.js App Router架构
text
app/
├── (auth)/
│   ├── login/page.tsx      # 客户端组件
│   └── register/page.tsx   # 客户端组件
├── (dashboard)/
│   ├── page.tsx            # 地图主页（客户端组件）
│   ├── profile/
│   │   └── page.tsx        # 个人中心（客户端组件）
│   └── layout.tsx          # 布局（服务端组件）
├── api/
│   └── ai/
│       ├── tags/route.ts   # API Route
│       ├── semantic/route.ts
│       └── music/route.ts
├── layout.tsx              # 根布局（服务端组件）
└── middleware.ts           # 路由保护
8.3 数据流模式
点位添加流程:

用户在ReviewForm提交 → 调用高德POI搜索API

用户选择店铺 → 表单自动填充

用户点击“AI推荐标签” → fetch('/api/ai/tags')

用户上传图片 → Supabase Storage上传 → 返回URL

用户保存 → Supabase Client插入reviews表

TanStack Query使缓存失效 → 地图更新

邀请协助流程:

用户A发起邀请 → 插入collaborations表

Supabase Realtime推送 → 用户B订阅

用户B填写评价 → 插入reviews表

更新collaborations表status='completed'

双方前端刷新共同回忆数据

9. 安全和权限考虑
9.1 Next.js Middleware路由保护
typescript
// middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs';
import { NextResponse } from 'next/server';

export async function middleware(req: NextRequest) {
  const res = NextResponse.next();
  const supabase = createMiddlewareClient({ req, res });
  const { data: { session } } = await supabase.auth.getSession();
  
  // 保护所有页面除了登录/注册
  if (!session && !req.nextUrl.pathname.startsWith('/auth')) {
    return NextResponse.redirect(new URL('/auth/login', req.url));
  }
  return res;
}
9.2 RLS策略（同前文）
9.3 密钥管理
Key	环境变量	访问位置
高德API Key	NEXT_PUBLIC_AMAP_KEY	前端
千问API Key	DASHSCOPE_API_KEY	API Routes（服务器端）
Supabase URL	NEXT_PUBLIC_SUPABASE_URL	前端+后端
Supabase Anon Key	NEXT_PUBLIC_SUPABASE_ANON_KEY	前端
10. 非功能性技术期望
10.1 性能目标
地图首屏加载: ≤2秒

AI标签推荐: ≤3秒

月度统计查询: ≤1秒

打卡记录列表分页: ≤300ms

Next.js页面切换: 软导航，无白屏

10.2 Next.js优化
图片使用next/image组件（自动优化）

地图组件动态导入（next/dynamic，避免SSR）

API Routes使用边缘运行时（Edge Runtime）降低延迟

10.3 可靠性期望
Supabase免费SLA: 99.9%可用性

高德地图API降级方案

千问API超时5秒降级

10.4 可观测性
Vercel Analytics: Web Vitals监控

Supabase Dashboard: 数据库监控

11. 交付风险和权衡
风险	缓解措施
Next.js SSR与高德地图不兼容	动态导入地图组件，禁用SSR
API Routes冷启动延迟	使用Edge Runtime，配置无服务器函数保持活跃
Supabase实时订阅连接不稳定	降级为轮询（每30秒）
千问API Key泄露	仅存储在服务器环境变量，API Routes代理
图片存储超1GB	前端限制单张≤2MB，提示用户压缩
12. 建议的构建顺序
阶段1: 基础框架

初始化Next.js + TypeScript + Tailwind项目

配置Supabase项目，启用Auth + Database + Storage

配置高德API Key（域名白名单）

实现登录/注册页面（客户端组件）

配置Next.js Middleware路由保护

配置shadcn/ui组件库

阶段2: 地图与点位CRUD
7. 实现地图页面（动态导入，禁用SSR）
8. 实现高德POI搜索组件
9. 实现点位添加表单（React Hook Form + Zod）
10. 实现AI标签推荐API Route
11. 实现图片上传（Supabase Storage + next/image）
12. 实现reviews表CRUD + TanStack Query缓存

阶段3: 社交功能
13. 实现好友系统（Supabase表 + 查询）
14. 实现邀请协助（collaborations表 + Realtime订阅）
15. 实现店铺详情卡片 + “共同去过”Tab

阶段4: 个人中心与总结
16. 实现打卡记录列表
17. 实现共同回忆模块
18. 实现月度总结模块
19. 实现语义搜索API Route + 前端组件

阶段5: 测试与上线
20. 手动测试所有验收标准
21. 配置Vercel环境变量
22. 配置自定义域名，启用HTTPS
23. 添加PWA manifest（next-pwa）
24. 发布MVP

13. 未决问题
Next.js渲染策略: 地图页面使用客户端组件（'use client'），其他页面可混用SSR？

建议: MVP全部使用客户端组件，简化开发，后续优化

API Routes运行时: 使用Node.js运行时还是Edge Runtime？

建议: Edge Runtime（冷启动快，适合AI代理）

Supabase客户端: 使用@supabase/auth-helpers-nextjs还是标准@supabase/supabase-js？

建议: 标准客户端 + 手动管理session（更灵活）

附录: 关键文件结构

text
tastebridge/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (dashboard)/
│   │   ├── page.tsx           # 地图主页
│   │   ├── profile/
│   │   │   └── page.tsx       # 个人中心
│   │   └── layout.tsx
│   ├── api/
│   │   └── ai/
│   │       ├── tags/
│   │       │   └── route.ts
│   │       ├── semantic/
│   │       │   └── route.ts
│   │       └── music/
│   │           └── route.ts
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── map/
│   ├── review/
│   ├── social/
│   ├── search/
│   ├── profile/
│   └── ui/                # shadcn/ui组件
├── hooks/
│   ├── useAuth.ts
│   ├── useReviews.ts
│   └── useRealtime.ts
├── lib/
│   ├── supabase/
│   │   ├── client.ts      # 浏览器客户端
│   │   └── server.ts      # 服务器客户端
│   └── utils.ts
├── stores/
│   └── appStore.ts
├── types/
│   └── index.ts
├── middleware.ts
├── next.config.js
├── tailwind.config.js
└── package.json
环境变量配置:

bash
# .env.local（本地开发）
NEXT_PUBLIC_SUPABASE_URL=xxx
NEXT_PUBLIC_SUPABASE_ANON_KEY=xxx
NEXT_PUBLIC_AMAP_KEY=xxx
DASHSCOPE_API_KEY=sk-xxx  # 千问API Key，仅服务器端

# Vercel环境变量（生产环境）
# 同上，DASHSCOPE_API_KEY不暴露给客户端