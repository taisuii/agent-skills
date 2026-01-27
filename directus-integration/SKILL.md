---
name: directus-integration
description: Guide agents to integrate Directus headless CMS with multi-language translation support, including SDK setup, collection design, queries, filtering, pagination, and M2M relationships. Use when the user wants to add or modify Directus-based content management, multi-language content, or CMS integration in web applications.
---

# Directus 集成 Skill（多语言内容管理）

本 Skill 指导代理在各类项目中集成 Directus 作为 Headless CMS，重点关注**多语言翻译系统**的实现。参考本项目的实现（`src/lib/directus.ts` + `src/pages/read.tsx`），涵盖：

- Directus SDK 初始化与配置
- 多语言翻译表设计（主表 + 翻译表）
- Collection 关联查询（M2M、O2M）
- 多语言翻译查找与回退逻辑
- 分页、过滤、排序
- 项目隔离（多租户支持）

## 何时使用本 Skill

当用户出现以下需求时，应使用此 Skill：

- 接入 / 修改 Directus CMS
- 实现多语言内容管理（文章、分类、标签等）
- 查询 Directus 集合（Collection）并处理翻译
- 实现内容过滤、分页、搜索
- 处理 Directus 的关联关系（M2M / O2M / M2O）

## Directus 核心概念

### 关键配置字段

- **DIRECTUS_URL**：Directus 实例地址（如 `https://cms.example.com`）
- 可选：**Access Token**（如需要认证的私有内容）

推荐环境变量（本项目已采用）：

- `NEXT_PUBLIC_DIRECTUS_URL`（前端可访问的公开 URL）

### Directus 数据结构

Directus 使用 **Collection** 来组织数据，类似于数据库表。常见模式：

- **主表**：存储核心数据和语言无关的字段（如 `id`、`slug`、`status`）
- **翻译表**：存储多语言字段（如 `title`、`content`），通过 `languages_code` 关联语言
- **关联表**（Junction Table）：处理多对多关系（如文章与分类、文章与标签）

### 本项目中的数据模型

本项目实现了一个多语言博客系统，包括：

#### 1. Articles（文章）

主表字段：
- `id`：主键
- `slug`：URL 友好标识符
- `status`：发布状态（`published` / `draft` / `archived`）
- `translations`：关联到翻译表（O2M）
- `categories`：关联到分类（M2M）
- `tags`：关联到标签（M2M）
- `project`：关联到项目（M2O），实现多租户隔离

翻译表 `articles_translations`：
- `id`
- `articles_id`：外键指向主表
- `languages_code`：语言代码（如 `zh-CN`、`en-US`）
- `title`：标题（翻译）
- `content`：内容（翻译）

#### 2. Categories（分类）

主表字段：
- `id`
- `slug`
- `translations`

翻译表 `categories_translations`：
- `id`
- `categories_id`
- `languages_code`
- `name`：分类名称（翻译）
- `description`：分类描述（翻译，可选）

#### 3. Tags（标签）

类似 Categories，也有主表和翻译表。

#### 4. Project（项目）

用于多租户隔离：
- `id`
- `slug`：项目标识（如 `sngzs`）

## 多语言翻译系统设计原则

### 设计模式：主表 + 翻译表

**主表**存储：
- 唯一标识（`id`、`slug`）
- 语言无关的元数据（状态、日期、关联）

**翻译表**存储：
- 与主表的关联（外键）
- 语言代码（`languages_code`）
- 可翻译字段（标题、内容、描述等）

这种模式的优势：
- 新增语言无需修改表结构
- 查询灵活：可按语言过滤
- 回退机制：主语言缺失时使用备用语言

### 翻译查找逻辑

本项目实现了通用翻译查找函数（见 `src/lib/directus.ts`）：

1. 优先查找目标语言的翻译
2. 若不存在，查找回退语言（如 `zh-CN`）
3. 若仍不存在，返回第一个可用翻译
4. 若完全无翻译，返回 `null`

```typescript
// 伪代码
function findTranslation(translations, targetLang, fallbackLang) {
  const target = translations.find(t => t.languages_code === targetLang);
  if (target) return target;
  
  const fallback = translations.find(t => t.languages_code === fallbackLang);
  if (fallback) return fallback;
  
  return translations[0] || null;
}
```

## 通用集成流程（各语言通用）

在任意项目中接入 Directus：

1. **在 Directus 中设计 Collection**
   - 创建主表和翻译表
   - 配置 O2M 关系（主表 → 翻译表）
   - 配置 M2M 关系（如文章 ↔ 分类，需要 Junction Table）
   - 配置权限（Public 角色可读取已发布内容）
2. **在应用中配置环境变量**
   - `NEXT_PUBLIC_DIRECTUS_URL` 或等价变量
3. **引入对应语言的 Directus SDK**
   - JavaScript / TypeScript：`@directus/sdk`
   - Python：`py-directus`
   - Java / Go / .NET：使用 REST API 或社区 SDK
4. **初始化 Directus 客户端**
   - 创建客户端实例
   - 配置 REST 或 GraphQL 适配器
5. **定义 TypeScript 类型 / 数据模型**（TypeScript 项目）
   - 为主表、翻译表定义接口
   - 定义关联数据的类型
6. **实现查询函数**
   - 查询列表：支持过滤、分页、排序
   - 查询单项：按 `id` 或 `slug`
   - 深度查询：嵌套关联数据（`fields` 参数）
7. **实现翻译查找辅助函数**
   - 根据语言代码查找翻译
   - 实现回退逻辑
8. **在前端 / 后端使用**
   - SSR：在 `getServerSideProps` / `getStaticProps` 中查询
   - CSR：在组件中调用 API 或直接查询（需配置 CORS）

## 本项目实现中可复用的模式

### 1. 初始化 Directus 客户端（Node.js / TypeScript）

在 `src/lib/directus.ts` 中：

```typescript
import { createDirectus, rest } from '@directus/sdk';

const DIRECTUS_URL = process.env.NEXT_PUBLIC_DIRECTUS_URL || 'http://localhost:8055';
const directus = createDirectus(DIRECTUS_URL).with(rest());
```

**要点**：
- 使用 `createDirectus` 创建客户端
- 链式调用 `.with(rest())` 添加 REST 适配器
- 从环境变量读取 URL，提供默认值

### 2. 定义 TypeScript 类型

为主表、翻译表、关联数据定义接口：

```typescript
export interface Translation {
  id: number;
  articles_id: number;
  languages_code: string;
  title: string;
  content: string;
}

export interface Article {
  id: number;
  slug: string;
  status?: string;
  translations?: Translation[];
  categories?: Category[];
  tags?: Tag[];
  project?: Array<{ id: number }>;
}
```

**要点**：
- 主表接口包含关联字段（数组类型）
- 翻译表接口包含外键和语言代码
- 可选字段使用 `?`

### 3. 查询列表（支持过滤、分页）

```typescript
export const getArticles = async (
  projectId: number,
  options?: { page?: number; limit?: number; categoryId?: number; tagId?: number }
) => {
  const page = options?.page || 1;
  const limit = options?.limit || 10;
  const offset = (page - 1) * limit;

  const filter: any = {
    _and: [
      { status: { _eq: 'published' } },
      { project: { id: { _eq: projectId } } },
    ],
  };

  if (options?.categoryId) {
    filter._and.push({
      categories: { categories_id: { _eq: options.categoryId } },
    });
  }

  const result = await directus.request(
    readItems('articles', {
      filter,
      fields: [
        'id',
        'slug',
        'status',
        'translations.*',
        'categories.*',
        'tags.*',
      ],
      sort: ['-id'],
      limit,
      offset,
    })
  );

  return { articles: result, total: result.length };
};
```

**要点**：
- 使用 `readItems` 查询集合
- `filter` 构建复杂过滤条件（`_and`、`_eq`、`_in` 等）
- `fields` 指定返回字段，使用 `.*` 获取关联数据的所有字段
- `sort` 排序（`-id` 降序）
- `limit` 和 `offset` 实现分页

### 4. 处理 M2M 关系的嵌套查询

Directus SDK 的 M2M 嵌套查询可能不完整，需要额外处理：

```typescript
// 先获取完整的分类和标签列表
const [fullCategories, fullTags] = await Promise.all([
  getCategories(projectId),
  getTags(projectId),
]);

// 创建 Map 用于快速查找
const categoryMap = new Map(fullCategories.map((cat) => [cat.id, cat]));

// 填充文章的关联数据
const articlesWithData = articles.map((article) => ({
  ...article,
  categories_data: article.categories
    ?.map((rel) => categoryMap.get(rel.categories_id))
    .filter(Boolean),
}));
```

**要点**：
- M2M 关系通过 Junction Table，字段名为 `{relation_name}_id`
- 先单独查询关联表，再手动填充
- 使用 `Map` 提高查找效率

### 5. 多语言翻译查找

```typescript
const findTranslation = <T extends Translation>(
  translations: T[] | undefined,
  language: string,
  fallbackLanguage: string = 'zh-CN'
): T | null => {
  if (!translations || translations.length === 0) return null;

  const translation = translations.find((t) => t.languages_code === language);
  if (translation) return translation;

  const fallback = translations.find((t) => t.languages_code === fallbackLanguage);
  return fallback || translations[0] || null;
};

export const getArticleTranslation = (article: Article, language: string) => {
  return findTranslation(article.translations, language);
};
```

**要点**：
- 泛型函数，支持不同类型的翻译
- 三层回退：目标语言 → 备用语言 → 第一个翻译
- 导出特定类型的翻译查找函数

### 6. 项目隔离（多租户）

```typescript
export const getProjectId = async (projectName: string): Promise<number | null> => {
  const result = await directus.request(
    readItems('project', {
      filter: { slug: { _eq: projectName } },
      limit: 1,
      fields: ['id'],
    })
  );
  return result.length > 0 ? result[0].id : null;
};
```

然后在查询时过滤：

```typescript
filter: {
  _and: [
    { project: { id: { _eq: projectId } } },
    // 其他条件...
  ],
}
```

**要点**：
- 通过 `slug` 查找项目 ID
- 在所有查询中添加项目过滤
- 实现数据隔离

### 7. 前端使用（React / Next.js）

```typescript
// 在 getServerSideProps 中查询
export const getServerSideProps: GetServerSideProps = async (context) => {
  const projectId = await getProjectId('sngzs');
  const { articles, total } = await getArticles(projectId, { page: 1, limit: 10 });
  
  return {
    props: { articles, total },
  };
};

// 在组件中使用
function ArticleList({ articles }: { articles: Article[] }) {
  const currentLanguage = 'zh-CN';
  
  return (
    <div>
      {articles.map((article) => {
        const translation = getArticleTranslation(article, currentLanguage);
        return (
          <div key={article.id}>
            <h2>{translation?.title}</h2>
            <p>{translation?.content}</p>
          </div>
        );
      })}
    </div>
  );
}
```

## 多语言 / 多后端移植建议

虽然本项目使用 Node.js / TypeScript，但 Directus 集成模式可以迁移到其它语言：

### Python（Flask / FastAPI / Django）

使用 `py-directus` 或直接调用 REST API：

```python
import requests

DIRECTUS_URL = os.getenv('DIRECTUS_URL', 'http://localhost:8055')

def get_articles(project_id, page=1, limit=10):
    offset = (page - 1) * limit
    params = {
        'filter': {
            '_and': [
                {'status': {'_eq': 'published'}},
                {'project': {'id': {'_eq': project_id}}}
            ]
        },
        'fields': ['id', 'slug', 'translations.*'],
        'limit': limit,
        'offset': offset,
        'sort': ['-id']
    }
    
    response = requests.get(
        f'{DIRECTUS_URL}/items/articles',
        params={'filter': json.dumps(params['filter']), 'fields': ','.join(params['fields'])},
    )
    
    return response.json()['data']

def find_translation(translations, language, fallback='zh-CN'):
    for trans in translations:
        if trans.get('languages_code') == language:
            return trans
    
    for trans in translations:
        if trans.get('languages_code') == fallback:
            return trans
    
    return translations[0] if translations else None
```

**要点**：
- 使用 `requests` 调用 REST API
- 过滤器需要 JSON 序列化
- 翻译查找逻辑完全复用

### Java（Spring Boot）

使用 `RestTemplate` 或 `WebClient`：

```java
@Service
public class DirectusService {
    @Value("${directus.url}")
    private String directusUrl;
    
    private final RestTemplate restTemplate;
    
    public List<Article> getArticles(int projectId, int page, int limit) {
        String url = directusUrl + "/items/articles";
        
        UriComponentsBuilder builder = UriComponentsBuilder.fromHttpUrl(url)
            .queryParam("filter[_and][0][status][_eq]", "published")
            .queryParam("filter[_and][1][project][id][_eq]", projectId)
            .queryParam("fields", "id,slug,translations.*")
            .queryParam("limit", limit)
            .queryParam("offset", (page - 1) * limit)
            .queryParam("sort", "-id");
        
        ResponseEntity<DirectusResponse> response = restTemplate.getForEntity(
            builder.toUriString(),
            DirectusResponse.class
        );
        
        return response.getBody().getData();
    }
    
    public Translation findTranslation(List<Translation> translations, String language, String fallback) {
        return translations.stream()
            .filter(t -> t.getLanguagesCode().equals(language))
            .findFirst()
            .orElseGet(() -> translations.stream()
                .filter(t -> t.getLanguagesCode().equals(fallback))
                .findFirst()
                .orElse(translations.isEmpty() ? null : translations.get(0))
            );
    }
}
```

### Go

使用 `net/http` 或 `go-directus` SDK：

```go
type DirectusClient struct {
    baseURL string
    client  *http.Client
}

func (c *DirectusClient) GetArticles(projectID int, page int, limit int) ([]Article, error) {
    offset := (page - 1) * limit
    
    filter := map[string]interface{}{
        "_and": []map[string]interface{}{
            {"status": map[string]string{"_eq": "published"}},
            {"project": map[string]map[string]int{"id": {"_eq": projectID}}},
        },
    }
    
    filterJSON, _ := json.Marshal(filter)
    
    url := fmt.Sprintf("%s/items/articles?filter=%s&fields=id,slug,translations.*&limit=%d&offset=%d&sort=-id",
        c.baseURL, url.QueryEscape(string(filterJSON)), limit, offset)
    
    resp, err := c.client.Get(url)
    // 处理响应...
}

func FindTranslation(translations []Translation, language string, fallback string) *Translation {
    for _, t := range translations {
        if t.LanguagesCode == language {
            return &t
        }
    }
    
    for _, t := range translations {
        if t.LanguagesCode == fallback {
            return &t
        }
    }
    
    if len(translations) > 0 {
        return &translations[0]
    }
    
    return nil
}
```

### .NET（ASP.NET Core）

使用 `HttpClient`：

```csharp
public class DirectusService
{
    private readonly HttpClient _httpClient;
    private readonly string _directusUrl;
    
    public async Task<List<Article>> GetArticlesAsync(int projectId, int page, int limit)
    {
        var offset = (page - 1) * limit;
        
        var filter = new
        {
            _and = new[]
            {
                new { status = new { _eq = "published" } },
                new { project = new { id = new { _eq = projectId } } }
            }
        };
        
        var filterJson = JsonSerializer.Serialize(filter);
        
        var url = $"{_directusUrl}/items/articles?filter={Uri.EscapeDataString(filterJson)}&fields=id,slug,translations.*&limit={limit}&offset={offset}&sort=-id";
        
        var response = await _httpClient.GetFromJsonAsync<DirectusResponse<Article>>(url);
        return response?.Data ?? new List<Article>();
    }
    
    public Translation FindTranslation(List<Translation> translations, string language, string fallback = "zh-CN")
    {
        return translations.FirstOrDefault(t => t.LanguagesCode == language)
            ?? translations.FirstOrDefault(t => t.LanguagesCode == fallback)
            ?? translations.FirstOrDefault();
    }
}
```

## 为代理编写 / 修改 Directus 集成代码的操作指引

当用户让你"接入 Directus"或"实现多语言内容"时，请按以下流程操作：

1. **识别应用类型与语言**
   - 判断是 Node.js / Python / Java / Go / .NET 等。
2. **定位配置来源**
   - 查找 `.env` 等配置文件，复用已有变量名（如 `DIRECTUS_URL`）。
3. **设计 / 检查 Directus Collection**
   - 主表：核心字段 + 关联
   - 翻译表：外键 + `languages_code` + 可翻译字段
   - 关联表：M2M Junction Table
4. **初始化 Directus 客户端**
   - 根据语言选择 SDK 或 REST API。
5. **实现查询函数**
   - 列表查询：支持过滤、分页、排序、深度查询
   - 单项查询：按 `id` 或 `slug`
   - 关联查询：处理 M2M / O2M 数据填充
6. **实现翻译查找逻辑**
   - 目标语言 → 回退语言 → 第一个翻译
   - 封装为辅助函数。
7. **在前端 / 后端使用**
   - SSR：在服务器端查询
   - CSR：通过 API 或直接查询（配置 CORS）
8. **处理语言代码的变体**
   - Directus 可能返回 `languages_code` 或 `Languages_code`（大小写）
   - 可能是字符串或对象（`{ code: 'zh-CN' }`）
   - 使用辅助函数标准化

## 输出格式建议

为用户生成 Directus 集成说明或示例代码时：

- **先给数据模型设计**（Collection 结构、字段、关系）。
- 再给查询函数示例（过滤、分页、关联）。
- 最后给翻译查找逻辑（回退机制）。
- 避免硬编码 URL，统一从环境变量读取。
- 提供 TypeScript 类型定义（如适用）。
