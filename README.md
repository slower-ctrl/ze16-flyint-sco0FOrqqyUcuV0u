
## 前言


大语言模型（Large Language Models, LLMs）近年来在各行各业中展现出了巨大的潜力和影响力。从自然语言处理到自动化客服，从内容生成到智能助手，LLMs正在改变我们与技术互动的方式。随着技术的不断进步，LLMs的应用场景也在不断扩展，成为未来发展的重要趋势。这篇文章将介绍如何使用WinUI（WASDK）和BotSharp开发一个多智能体桌面机器人管理助手，展示LLMs在实际应用中的强大功能和广阔前景。


## 技术介绍


### .NET


[.NET](https://github.com) 是免费的、开源的、跨平台的框架，用于构建新式应用和强大的云服务。


![.NET](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126162300305-1617249552.png)


### WinUI（WASDK）


[Windows 应用 SDK](https://github.com) 是一组新的开发人员组件和工具，它们代表着 Windows 应用开发平台的下一步发展。 Windows 应用 SDK 提供一组统一的 API 和工具，可供从 Windows 11 到 Windows 10 版本 1809 上的任何桌面应用以一致的方式使用。


Windows 应用 SDK 不会用 C\+\+ 替换 Windows SDK 或现有桌面 Windows 应用类型，例如 .NET（包括 Windows 窗体和 WPF）和桌面 Win32。 相反，Windows 应用 SDK 使用一组通用 API 来补充这些现有工具和应用类型，开发人员可以在这些平台上依赖这些 API 来执行操作。 有关更多详细信息，请参阅 Windows 应用 SDK 的优势。


![wasdk](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126162536524-336613766.png)


### BotSharp


[BotSharp](https://github.com) 是一个开源应用程序框架，可加快将 LLM 集成到您当前的业务系统中的速度。本项目涉及自然语言理解和音频处理技术，旨在推动智能机器人助手在信息系统中的开发和应用。开箱即用的机器学习算法使普通程序员能够更快、更轻松地开发人工智能应用程序。


BotSharp 是一个高度兼容且高度可扩展的平台构建器。它严格按照组件原则，将平台构建器中需要的每个部分解耦。因此，您可以选择不同的 UI/UX，或者选择不同的 NLP 标记器，或者选择更高级的算法来执行 NER 任务。它们都是基于未加密的接口进行调制的。


![BotSharp](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126162210700-405597762.png)


### 大语言模型的函数调用（这个是理解BotSharp框架的核心知识点）


**函数调用允许您将模型连接到外部工具和系统。这对于许多事情都很有用，例如为 AI 助手提供功能，或在应用程序和模型之间构建深度集成。**


**[openai官方文档函数调用介绍文档](https://github.com)**


## 助手功能介绍


助手名为[**电子脑壳**](https://github.com)本身是负责开源硬件ElectronBot桌面机器人和瀚文键盘的操作配置。


新版本重构方向是深度集成多智能体交互的能力，目前新版本重点优化功能如下：


* 增强对话能力，添加大语言模型对话能力。
* 增强交互，添加生图和自然语言理解进行硬件控制，例如生图之后直接设置到桌面机器人屏幕上或者键盘屏幕上。
* 增强语音对话能力。
* 以前硬编码的逻辑，现在都可以通过大语言模型的函数调用进行语义化理解，更灵活。


![electronbot](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126181310068-1533723150.jpg)


目前软件还在开发中，但是BotSharp和大语言模型交互的功能已经开发差不多了，所以编写这篇博客记录一下。


**博客演示的代码是在电子脑壳源码的dev分支。**


目前文字大模型使用的是阿里的通义千问2\.5 72b（qwen2\.5\-72b\-instruct）社区开源版本，图片大模型使用的是通义万相（wanx\-v1）。


可以通过聊天进行天气查询，开关等，以及学单词，生图片等等其他功能，这些功能可以和上图的一些机器人进行互动。


演示效果如下：
![yanshi](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126183945926-61399501.gif)


## 代码实现过程


### 1\. 实现BotSharp的LiteDB存储


做这个实现的原因是我想替换掉框架本身默认的文件存储，因为我是开发桌面程序，所以mongodb这类的数据库也不在考虑范围，LiteDB也是文档数据库，使用上也比较简单，就作为数据存储的选项了。而且原本的软件的数据也可以都迁移到LiteDB上，算是统一了一些。


![LiteDB](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126163433461-874482188.png)


[源码我fork到我的名下了修改代码在litedb分支](https://github.com):[FlowerCloud机场](https://yunbeijia.com)


### 2\. 针对OpenAI插件进行改造


做这个操作的原因是为了兼容国内的大语言模型，有些时候OpenAI访问不了，可以通过国内的一些模型进行替换，例如智普清言，通义千问，以及讯飞的一些模型。


通过代码兼容自定义Endpoint，就可以随意切换兼容的模型了。


代码段如下防止图挂了：



```
    public static OpenAIClient GetClient(string provider, string model, IServiceProvider services)
    {
        var settingsService = services.GetRequiredService();
        var settings = settingsService.GetSetting(provider, model);

        var options = string.IsNullOrEmpty(settings.Endpoint)
            ? null
            : new OpenAIClientOptions { Endpoint = new Uri(settings.Endpoint) };

        return new OpenAIClient(new ApiKeyCredential(settings.ApiKey), options);
    }


```

![OpenAI](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126163937208-1584641650.png)


### 3\. 基于核心模块编写UI代码


BotSharp本身的demo是基于web服务编写的，有一套webui和一套封装好的api，但是我是基于桌面程序编写的，所以我就借鉴了社区一些开源的软件的代码，以及一些设计理念，整合了一个简单的聊天UI，针对发送消息，聊天列表，以及生成产物的保存等。


最左边的是机器人功能区域，中间为聊天区域，右侧为灵犀空间，生成的图片，单词以及天气内容都会保存一下，便于后期的查找。


![img](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126165008808-583910836.png)


### 4\. 功能模块的智能体代码


代码目录结构如下：


![agent](https://img2023.cnblogs.com/blog/1690009/202411/1690009-20241126184647869-2011798208.png)


以生图函数为例 下面是传给大模型的生图函数定义



```
{
  "name": "custom_generate_image",
  "description": "如果用户想生成图片可以调用此方法进行图片生成。",
  "parameters": {
    "type": "object",
    "properties": {
      "image_name": {
        "type": "string",
        "description": "根据用户描述给图片起个名称。"
      },
      "image_description": {
        "type": "string",
        "description": "用户进行的图片描述。"
      }
    },
    "required": [ "image_description" ]
  }
}

```

关联的生图函数实现类，可以被大语言模型调用。



```
public class CustomGenerateImageFn : IFunctionCallback
{
    public string Name => "custom_generate_image"; //和json配置的函数名字匹配

    private readonly IServiceProvider _service;
    private readonly IBotToolService _botToolService;
    private readonly JsonSerializerOptions _options;
    private readonly ILingxiSpaceService _lingxiSpaceService;
    private readonly IConversationService _conversationService;
    public CustomGenerateImageFn(IServiceProvider service,
        IBotToolService botToolService,
        ILingxiSpaceService lingxiSpaceService,
        IConversationService conversationService)
    {
        _service = service;
        _options = new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true,
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            WriteIndented = true,
            AllowTrailingCommas = true,
            Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
        };
        _botToolService = botToolService;
        _lingxiSpaceService = lingxiSpaceService;
        _conversationService = conversationService;
    }

    public async Task<bool> Execute(RoleDialogModel message)
    {
        // 函数反序列化之后的参数
        var args = JsonSerializer.Deserialize(message.FunctionArgs ?? "", _options) ?? new CustomGenerateImageFunctionArgs();

        message.StopCompletion = true;

        var clientFactory = _service.GetRequiredService();
        using var httpClient = clientFactory.CreateClient();
        var llmProviderService = _service.GetRequiredService();
        var model = llmProviderService.GetSetting("tongyi", "wanx-v1");
        if (model == null)
        {
            return false;
        }
        var request = new GenerateImageRequest
        {
            Model = "wanx-v1",
            Input = new GenerateImageInput
            {
                Prompt = args.ImageDescription
            },
            Parameters = new GenerateImageParameters
            {
                Style = "",
                Size = "1024*1024",
                N = 1
            }
        };
        var generateImageUrl = $"{model.Endpoint.TrimEnd('/')}/services/aigc/text2image/image-synthesis";

        // 添加认证头部请求头
        httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", model.ApiKey);
        httpClient.DefaultRequestHeaders.Add("X-DashScope-Async", "enable");
        var result = await httpClient.PostAsJsonAsync(generateImageUrl, request);
        if (!result.IsSuccessStatusCode)
        {
            return false;
        }
        var taskContent = await result.Content.ReadAsStringAsync();
        var resultData = JsonSerializer.Deserialize(taskContent, _options);

        var taskUrl = $"{model.Endpoint.TrimEnd('/')}/tasks/{resultData?.Output.TaskId}";

        var maxRetries = 5;
        var retryCount = 0;

        while (retryCount < maxRetries)
        {
            var taskResult = await httpClient.GetAsync(taskUrl);
            if (!taskResult.IsSuccessStatusCode)
            {
                return false;
            }
            var taskResultContent = await taskResult.Content.ReadAsStringAsync();
            var taskResponse = JsonSerializer.Deserialize(taskResultContent, _options);
            if (taskResponse?.Output.TaskStatus == "SUCCEEDED")
            {
                var url = taskResponse?.Output.Results.FirstOrDefault()?.Url;

                if (string.IsNullOrEmpty(url))
                {
                    return false;
                }

                // 下载图片并转换为Base64
                var imageBytes = await httpClient.GetByteArrayAsync(url);
                var base64Image = Convert.ToBase64String(imageBytes);

                var generateImageContent = new GenerateImageContent
                {
                    Name = args.ImageName,
                    Description = args.ImageDescription,
                    ImageData = $"data:{MediaTypeNames.Image.Png};base64,{base64Image}"
                };

                //保存生成的图片
                var lingxiSpace = await _lingxiSpaceService.AddAsync(new LingxiSpace
                {
                    Id = Guid.NewGuid().ToString(),
                    ConversationId = _conversationService.ConversationId,
                    Content = JsonSerializer.SerializeToDocument(generateImageContent, _options),
                    Name = args.ImageName,
                    Desc = args.ImageDescription,
                    Type = LingxiSpaceType.Image,
                    CreatedTime = DateTime.UtcNow
                });

                WeakReferenceMessenger.Default.Send(lingxiSpace);
                break;
            }
            await Task.Delay(10000); // 等待10秒后再次轮询
            retryCount++;
        }
        return retryCount < maxRetries;
    }

```

### 5\. 功能模块的加载


BotSharp采用插件模式开发，需要在配置中配置要加载的模块，然后项目启动就会加载模块注入服务。


目前我启用的模块配置如下：



```
  "PluginLoader": {
    "Assemblies": [
      "BotSharp.Core",
      "BotSharp.Logger",
      "BotSharp.Plugin.OpenAI",
      "BotSharp.Plugin.AzureOpenAI",
      "BotSharp.Plugin.MetaGLM",
      "BotSharp.Plugin.LiteDBStorage",
      "Verdure.Braincase.Copilot.Plugin"
    ]
  }

```

服务注入也很简单，主要是AddBotSharpCore的注入，BotSharp本身是有用户的概念的，所以我实现了一个BotUserIdentity做了用户的默认数据初始化，大家可以根据需要操作。



```
           // add botsharp
           .AddTransient()
           .AddTransient()
           .AddTransient()
           .AddTransient()
           .AddTransient()
           .AddBotSharpCore(config, options =>
           {
               options.JsonSerializerOptions.Converters.Add(new RichContentJsonConverter());
               options.JsonSerializerOptions.Converters.Add(new TemplateMessageJsonConverter());
           })
           .AddSingleton(dbSettings)
           .AddHttpContextAccessor()
           .AddScoped()
           .AddScoped()
           .AddScoped()
           .AddBotSharpLogger(config)

```

如果看到这里，大家还是一头雾水的话，可以多看看BotSharp的设计理念，当然如果有需要我可以再写一篇BotSharp的讲解文章。


## 心得体会


随着大模型能力的提升，大模型的应用场景也会越来越多，以后的大模型应该会作为基础设施供人们使用，基于大模型进行开发的岗位应该会越来越多，感觉大模型真的是生产力工具，我最近在开发这些功能的时候，也会借助Github Copilot进行一些功能的开发，效率高很多。


希望在未来人类是驾驭AI，而不是被AI给取代了。


## 参考推荐文档项目如下：


* [电子脑壳源码地址](https://github.com)
* [BotSharp 文档](https://github.com)
* [智能体开发框架 BotSharp源码](https://github.com)
* [聊天界面参考的项目 Rodel Agent](https://github.com)


