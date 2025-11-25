基于 **Unreal Engine 5（UE5）** 的数字人项目（如 Fay-UE5）核心是打通「自然语言交互→大模型推理→数字人动作 / 语音生成」的端到端流程，其代码实现和技术架构围绕 **UE5 引擎特性**（如蓝图、C++、MetaHuman、动画系统）与 **大模型服务**（如 ChatGPT、本地化 LLM）的协同展开。以下从「整体架构、核心模块代码实现、大模型调用流程、动作与回复联动」四个维度详细拆解：

[UE5 读写本地JSON，发送HTTP请求（get）_ue与json通信-CSDN博客](https://blog.csdn.net/qq_51654332/article/details/131489044)

## 一、整体技术架构

Fay-UE5 这类数字人项目的核心是「感知→决策→表达」的闭环，架构上分为 5 个关键模块，各模块通过 UE5 的 **事件驱动**（蓝图 Event）或 **C++ 接口** 串联：

| 模块               | 功能                                                         | 技术依赖                                             |
| ------------------ | ------------------------------------------------------------ | ---------------------------------------------------- |
| **输入交互模块**   | 接收用户输入（语音 / 文本 / 手势），转换为机器可处理的信号   | UE5 蓝图、麦克风输入组件、Text Widget                |
| **大模型调用模块** | 将用户输入传递给大模型（本地 / 云端），获取语义回复和情绪 / 动作指令 | C++ HTTP 客户端、大模型 API（OpenAI / 本地化 LLaMA） |
| **动作驱动模块**   | 根据大模型输出的动作指令，控制数字人骨骼动画（表情 / 肢体）  | UE5 动画蓝图、MetaHuman 骨骼系统、LiveLink           |
| **语音合成模块**   | 将大模型的文本回复转换为语音音频，驱动数字人嘴型同步         | TTS 服务（Azure TTS / 讯飞 TTS）、UE5 音频组件       |
| **状态管理模块**   | 维护数字人状态（ idle / 对话中 / 动作中），避免动作冲突和交互卡顿 | C++ 状态机、蓝图变量（如 `bIsTalking`）              |

注：Fay-UE5 基于 UE5 的 **MetaHuman**（数字人资产）和 **Animation Blueprints**（动画控制）构建，核心是通过代码将「大模型的语义输出」映射为「数字人的多模态表达」。

## 二、核心模块代码实现（以 Fay-UE5 为例）

以下聚焦「大模型调用」「动作驱动」「语音 - 动作联动」三个核心模块，结合 UE5 的 **C++ 与蓝图混合开发** 模式（C++ 负责底层逻辑，蓝图负责可视化编辑）讲解。

### 1. 大模型调用模块：连接 UE5 与 AI 大脑

数字人无法直接运行大模型（UE5 不原生支持大模型推理），需通过「API 调用」或「本地化推理服务」获取回复。Fay-UE5 常用 **HTTP 客户端** 调用云端大模型（如 ChatGPT），或通过 **TCP 协议** 连接本地部署的 LLM（如 LLaMA 2）。

#### （1）C++ 核心代码：HTTP 调用大模型（以 OpenAI API 为例）

首先在 UE5 中创建 C++ 类（继承 `UObject`），封装 HTTP 请求逻辑，暴露函数给蓝图调用：

```c++
// 头文件：DigitalHumanAI.h
#include "CoreMinimal.h"
#include "UObject/NoExportTypes.h"
#include "Http.h" // UE5 HTTP 模块
#include "DigitalHumanAI.generated.h"

UCLASS(Blueprintable, BlueprintType)
class FAYUE5_API UDigitalHumanAI : public UObject
{
    GENERATED_BODY()

public:
    // 蓝图可调用：发送用户输入给大模型，获取回复
    UFUNCTION(BlueprintCallable, Category = "DigitalHumanAI")
    void SendQueryToAI(const FString& UserInput, const FString& ConversationHistory);

private:
    // HTTP 请求完成回调
    void OnAIResponseReceived(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful);

    // 大模型 API 配置（可在蓝图中设置）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AIConfig", meta = (AllowPrivateAccess = "true"))
    FString OpenAI_APIKey = "sk-xxx"; // 你的 API Key

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AIConfig", meta = (AllowPrivateAccess = "true"))
    FString OpenAI_Model = "gpt-3.5-turbo"; // 模型版本
};

// 源文件：DigitalHumanAI.cpp
#include "DigitalHumanAI.h"
#include "Serialization/JsonSerializer.h" // JSON 解析

void UDigitalHumanAI::SendQueryToAI(const FString& UserInput, const FString& ConversationHistory)
{
    // 1. 构造 HTTP 请求
    TSharedRef<IHttpRequest, ESPMode::ThreadSafe> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL("https://api.openai.com/v1/chat/completions"); // OpenAI API 地址
    Request->SetVerb("POST");
    Request->SetHeader("Content-Type", "application/json");
    Request->SetHeader("Authorization", "Bearer " + OpenAI_APIKey);

    // 2. 构造请求体（JSON 格式，包含对话历史和用户输入）
    TSharedPtr<FJsonObject> RequestBody = MakeShareable(new FJsonObject());
    RequestBody->SetStringField("model", OpenAI_Model);
    
    TArray<TSharedPtr<FJsonValue>> Messages;
    // 添加对话历史（格式：[{"role":"user","content":"xxx"}, {"role":"assistant","content":"xxx"}]）
    if (!ConversationHistory.IsEmpty())
    {
        TSharedPtr<FJsonObject> HistoryMsg = MakeShareable(new FJsonObject());
        HistoryMsg->SetStringField("role", "system");
        HistoryMsg->SetStringField("content", "对话历史：" + ConversationHistory);
        Messages.Add(MakeShareable(new FJsonValueObject(HistoryMsg)));
    }
    // 添加当前用户输入
    TSharedPtr<FJsonObject> UserMsg = MakeShareable(new FJsonObject());
    UserMsg->SetStringField("role", "user");
    UserMsg->SetStringField("content", UserInput);
    Messages.Add(MakeShareable(new FJsonValueObject(UserMsg)));
    
    RequestBody->SetArrayField("messages", Messages);

    // 3. 序列化 JSON 并发送请求
    FString RequestBodyString;
    TSharedRef<TJsonWriter<>> Writer = TJsonWriterFactory<>::Create(&RequestBodyString);
    FJsonSerializer::Serialize(RequestBody.ToSharedRef(), Writer);
    Request->SetContentAsString(RequestBodyString);

    // 绑定回调函数（请求完成后触发）
    Request->OnProcessRequestComplete().BindUObject(this, &UDigitalHumanAI::OnAIResponseReceived);
    Request->ProcessRequest();
}

void UDigitalHumanAI::OnAIResponseReceived(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful)
{
    if (!bWasSuccessful || !Response.IsValid())
    {
        UE_LOG(LogTemp, Error, TEXT("AI 请求失败！"));
        return;
    }

    // 1. 解析响应 JSON
    FString ResponseString = Response->GetContentAsString();
    TSharedPtr<FJsonObject> ResponseJson;
    TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(ResponseString);
    if (!FJsonSerializer::Deserialize(Reader, ResponseJson))
    {
        UE_LOG(LogTemp, Error, TEXT("JSON 解析失败！"));
        return;
    }

    // 2. 提取大模型回复文本（格式：choices[0].message.content）
    TArray<TSharedPtr<FJsonValue>> Choices = ResponseJson->GetArrayField("choices");
    if (Choices.Num() == 0)
    {
        UE_LOG(LogTemp, Error, TEXT("无回复结果！"));
        return;
    }
    FString AIReply = Choices[0]->AsObject()->GetObjectField("message")->GetStringField("content");

    // 3. 触发蓝图事件：传递回复文本给动作/语音模块
    OnAIReplyGenerated.Broadcast(AIReply); // 自定义蓝图事件
}
```

#### （2）蓝图调用逻辑

在 UE5 蓝图中创建「数字人控制器」蓝图，调用上述 C++ 函数：

1. 从用户输入组件（如 Text Widget 的「提交按钮」）获取文本；
2. 调用 `SendQueryToAI` 函数，传入用户输入和对话历史；
3. 绑定 `OnAIReplyGenerated` 事件，接收大模型回复文本，传递给「语音合成模块」和「动作驱动模块」。

### 2. 动作驱动模块：让数字人 “动起来”

Fay-UE5 基于 **MetaHuman**（UE5 官方数字人资产）的骨骼系统，通过「动画蓝图」控制骨骼动画（表情、肢体动作）。核心是：根据大模型输出的「动作指令」（如 “挥手”“微笑”“点头”），触发对应的动画片段。

#### （1）动作指令定义

大模型回复中需包含动作指令（可通过 Prompt 约束格式），例如：

- 用户输入：“你好”
- 大模型回复：“你好！[动作：微笑，时长：2s]”

通过 C++/ 蓝图解析回复文本中的动作指令（正则表达式匹配 `\[动作:(\w+), 时长:(\d+)s\]`）。

#### （2）动画蓝图逻辑（核心）

在 UE5 中创建「数字人动画蓝图」（继承 `UAnimationBlueprint`），通过「状态机」控制动画切换：

1. **基础状态**：`Idle`（待机动画，如呼吸、轻微头部晃动）；
2. **对话状态**：`Talking`（说话时的嘴型同步动画，通过 LiveLink 或音频驱动）；
3. **动作状态**：`Gesture`（手势动画，如挥手、点头，通过状态机切换）。

**蓝图关键节点**：

- **动画通知**：在 `Talking` 动画中添加「通知」，触发语音播放（确保嘴型与语音同步）；
- **状态切换**：通过蓝图变量（如 `CurrentGesture`）控制从 `Idle` 切换到 `Gesture`，播放对应动作动画；
- **动画融合**：使用 `Blend Space` 融合不同动作（如走路时挥手，避免动作生硬）。

#### （3）C++ 与蓝图联动

在 C++ 中定义动作控制函数，暴露给蓝图：

```c++
// 头文件：DigitalHumanAnimation.h
UCLASS()
class FAYUE5_API UDigitalHumanAnimation : public UAnimationBlueprintLibrary
{
    GENERATED_BODY()

public:
    // 蓝图可调用：触发指定动作
    UFUNCTION(BlueprintCallable, Category = "DigitalHumanAnimation")
    static void PlayGestureAnimation(UAnimInstance* AnimInstance, const FString& GestureName, float Duration);
};

// 源文件：DigitalHumanAnimation.cpp
void UDigitalHumanAnimation::PlayGestureAnimation(UAnimInstance* AnimInstance, const FString& GestureName, float Duration)
{
    if (!AnimInstance) return;

    // 假设动作动画的命名规范：Gesture_挥手、Gesture_微笑
    FName AnimName = *FString("Gesture_") + GestureName;
    AnimInstance->Montage_Play(AnimInstance->FindMontage(AnimName)); // 播放动作蒙太奇
}
```

1. 在蓝图中，解析大模型回复的动作指令后，调用 `PlayGestureAnimation` 函数，触发对应动作。

   ### 3. 语音 - 动作联动：同步回复与表达

   核心是「语音播放」与「动作执行」的时序同步：

   1. 大模型回复文本 → 传递给 TTS 服务 → 生成语音音频（如 WAV 文件）；
   2. UE5 加载音频文件，通过 `UAudioComponent` 播放；
   3. 播放音频的同时，触发「说话动画」（嘴型同步）和「手势动作」（根据回复语义）。

   #### （1）TTS 服务调用（以 Azure TTS 为例）

   类似大模型调用，通过 HTTP 请求将文本转换为语音音频：

```c++
// 简化代码：调用 Azure TTS API 生成语音
void UDigitalHumanTTS::GenerateSpeech(const FString& Text)
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL("https://eastus.tts.speech.microsoft.com/cognitiveservices/v1");
    Request->SetVerb("POST");
    Request->SetHeader("Ocp-Apim-Subscription-Key", "your-azure-key");
    Request->SetHeader("Content-Type", "application/ssml+xml");
    Request->SetHeader("X-Microsoft-OutputFormat", "riff-24khz-16bit-mono-pcm"); // 音频格式

    // 构造 SSML（语音合成标记语言，控制语速、语调）
    FString SSML = FString::Printf(TEXT(
        "<speak version='1.0' xmlns='http://www.w3.org/2001/10/synthesis' xml:lang='zh-CN'>"
        "<voice name='zh-CN-XiaoxiaoNeural'>%s</voice>"
        "</speak>"), *Text);
    Request->SetContentAsString(SSML);

    // 绑定回调：接收音频数据并播放
    Request->OnProcessRequestComplete().BindUObject(this, &UDigitalHumanTTS::OnSpeechGenerated);
    Request->ProcessRequest();
}

// 音频生成完成后，播放并触发嘴型动画
void UDigitalHumanTTS::OnSpeechGenerated(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful)
{
    if (bWasSuccessful && Response.IsValid())
    {
        // 保存音频数据到 SoundWave
        USoundWave* SpeechSound = NewObject<USoundWave>();
        SpeechSound->SetSoundWaveForm(Response->GetContent().GetData(), Response->GetContentLength());
        
        // 播放语音
        UAudioComponent* AudioComp = DigitalHumanActor->FindComponentByClass<UAudioComponent>();
        AudioComp->SetSound(SpeechSound);
        AudioComp->Play();

        // 触发说话动画（蓝图事件）
        OnSpeechStarted.Broadcast();
    }
}
```

#### （2）嘴型同步（Lip-Sync）

Fay-UE5 常用 **UE5 LiveLink** 或 **MetaHuman LipSync** 实现嘴型同步：

- **LiveLink**：通过外部摄像头捕捉真人嘴型，实时驱动数字人；
- **音频驱动**：在动画蓝图中使用 `Audio Spectrum Analyzer` 节点，分析语音音频的频率，驱动嘴型骨骼（如 jaw、lip 骨骼）。

## 三、大模型调用的关键优化

1. **对话历史管理**：在 C++ 中用 `FString` 或 `TArray` 存储对话历史，每次调用大模型时传入，实现多轮对话；
2. **本地化部署**：若需离线运行，可将 LLaMA 2/ChatGLM 部署为本地 HTTP 服务（如用 FastAPI 封装），UE5 调用本地服务地址（`http://localhost:8000/chat`）；
3. **请求缓存**：重复输入时缓存大模型回复，减少 API 调用次数；
4. **超时处理**：在 HTTP 请求中设置超时时间（如 5s），超时后触发重试或提示用户 “网络繁忙”。

## 四、总结：核心流程串联

1. **用户输入**：用户通过 Text Widget 输入文本（或麦克风语音转文字）；
2. **大模型推理**：UE5 调用 C++ HTTP 客户端，将输入 + 对话历史发送给大模型，获取回复文本（含动作指令）；
3. **动作触发**：解析回复中的动作指令，通过蓝图调用 `PlayGestureAnimation`，切换数字人动作；
4. **语音合成**：将回复文本传递给 TTS 服务，生成语音音频，UE5 播放音频并触发嘴型同步；
5. **状态重置**：语音和动作完成后，数字人回到 `Idle` 状态，等待下一次交互。

### **Q:在数字人实际部署后，有时候用户通过指令攻击，可能让数字人产生奇怪的行为，如何避免这种事件发生**

这是数字人部署后非常关键的安全问题，称为 **“Prompt Injection”（提示注入攻击）** 或 **“指令攻击”**。攻击者通过精心设计的输入，诱导数字人执行非预期的、可能有害的行为。要解决这个问题，需要从 **“输入过滤”、“指令规范化”、“行为约束”、“模型防御”** 等多个层面进行系统性防御。

### 一、问题根源分析

数字人产生奇怪行为的核心原因是：

1. **模型的 “指令遵循” 特性**：大模型会尽力遵循用户的指令，即使指令是恶意的。
2. **输入的 “模糊性”**：自然语言本身具有歧义，攻击者可以利用这一点绕过简单的关键词过滤。
3. **缺乏 “常识判断”**：模型可能无法识别某些指令在现实世界中的荒谬性或危害性。
4. **权限边界不清**：数字人可能被赋予了超出其应用场景的操作权限。

### 二、核心防御策略与技术实现

#### 1. 输入过滤与意图识别（第一道防线）

在将用户输入传递给大模型之前，进行严格的检查和过滤。

- **关键词与模式匹配**：
  - **实现方式**：维护一个动态更新的 “敏感指令” 或 “危险操作” 关键词库（如 “删除文件”、“格式化硬盘”、“访问系统”、“扮演黑客” 等）。
  - **技术**：使用正则表达式或字符串匹配算法进行快速检查。

- **意图分类（Intent Classification）**：
  - **实现方式**：在输入过滤阶段增加一个轻量级的 NLP 模型（如 BERT、RoBERTa 的微调模型），专门用于判断用户输入的 “意图” 是否属于数字人被允许的范围。
  - **分类**：将意图分为 “闲聊”、“任务查询”、“功能调用”、“恶意指令” 等。只有 “闲聊” 和 “任务查询” 等安全意图才能继续传递。
  - **部署**：可以将这个分类模型部署为一个独立的 API 服务，UE5 端通过 HTTP 请求进行调用。
  - **优势**：比简单的关键词过滤更智能，能理解上下文。

- **语义相似度检测**：
  - **实现方式**：将用户输入与已知的 “恶意指令样本库” 进行语义相似度比对。
  - **技术**：使用 Sentence-BERT 等模型将文本转换为向量，然后计算余弦相似度。如果相似度超过阈值，则判定为恶意输入。

#### 2. 指令规范化与 “角色固化”（第二道防线）

在将用户输入发送给大模型时，通过精心设计的 “System Prompt”（系统提示）来约束模型的行为。

- **清晰定义角色与能力边界**：
  - 在 System Prompt 中明确告诉大模型：
    - 它的角色是什么（例如：“你是一个友好的数字助手，专门回答关于 UE5 的技术问题”）。
    - 它能做什么，不能做什么（例如：“你只能提供信息，不能执行任何系统命令或访问用户的本地文件”）。
      - 当遇到无法回答或处理的请求时，应该如何回应（例如：“对不起，我无法提供这方面的帮助。”）。

#### 3. 行为约束与沙箱化（第三道防线）

即使模型生成了恶意指令，也要确保数字人无法真正执行。

- **动作与功能白名单**：
  - **实现方式**：为数字人定义一个明确的 “可执行动作列表” 或 “可调用功能列表”。
  - **技术**：在动作驱动模块中，所有动作的触发都必须经过白名单检查。

### 三、总结

防御数字人的指令攻击是一个系统性工程，单一的防御手段往往不够。最有效的策略是采取 **“纵深防御”（Defense in Depth）** 理念，将上述多种方法结合起来：

1. **前端过滤**：用关键词和意图分类快速拦截明显的恶意输入。
2. **中端规范**：用强大的 System Prompt 固化数字人的角色和行为边界。
3. **后端约束**：用动作白名单和沙箱化确保即使产生恶意指令也无法执行。
4. **模型优化**：长期来看，通过微调等方式从根源上提升模型的安全性。
5. **持续监控**：建立完善的监控和响应机制，以便及时发现和处理新的威胁。