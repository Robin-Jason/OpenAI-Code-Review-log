# OpenAi 代码评审.
### 😀代码评分：80
#### 😀代码逻辑与目的：
该代码片段展示了如何使用 OpenAI 进行代码审查的过程。`CodeReviewPromptProvider` 接口定义了一个方法 `buildPrompts`，用于根据提供的 diff 内容构建模型提示词。`DefaultCodeReviewPromptProvider` 类实现了这个接口，并提供了构建提示词的具体逻辑。`OpenAiCodeReviewService` 类则负责调用 OpenAI 的 API 进行代码审查，并将结果记录到日志中。

#### ✅代码优点：
1. 使用接口和实现类分离了提示词构建逻辑，符合单一职责原则。
2. `DefaultCodeReviewPromptProvider` 类提供了详细的注释，清晰地描述了其功能。

#### 🤔问题点：
1. `REVIEW_INSTRUCTION` 字符串过长，可能会导致性能问题，特别是在处理大量数据时。
2. `buildPrompts` 方法中直接添加了用户指令和 diff 内容，没有对指令进行格式化或清理，可能会影响模型的输出。
3. `OpenAiCodeReviewService` 类中的 `codeReview` 方法没有异常处理逻辑，如果模型调用失败，可能会导致服务崩溃。

#### 🎯修改建议：
1. 将 `REVIEW_INSTRUCTION` 字符串分割成更小的部分，以减少内存消耗和提高性能。
2. 在添加用户指令和 diff 内容之前，对其进行格式化和清理，例如去除无关的空白字符或特殊字符。
3. 在 `codeReview` 方法中添加异常处理逻辑，以处理模型调用失败的情况。

#### 💻修改后的代码：
```java
// DefaultCodeReviewPromptProvider.java
public class DefaultCodeReviewPromptProvider implements CodeReviewPromptProvider {

    private static final String USER_INSTRUCTION =
            "你是一位资深编程专家，拥有深厚的编程基础和广泛的技术栈知识。你的专长在于识别代码中的低效模式、安全隐患、以及可维护性问题，并能提出针对性的优化策略。你擅长以易于理解的方式解释复杂的概念，确保即使是初学者也能跟随你的指导进行有效改进。在提供优化建议时，你注重平衡性能、可读性、安全性、逻辑错误、异常处理、边界条件，以及可维护性方面的考量，同时尊重原始代码的设计意图。\n"
                    + "你总是以鼓励和建设性的方式提出反馈，致力于提升团队的整体编程水平，详尽指导编程实践，雕琢每一行代码至臻完善。用户会将仓库代码分支修改代码给你，以git diff 字符串的形式提供，你需要根据变化的代码，帮忙review本段代码。然后你review内容的返回内容必须严格遵守下面我给你的格式，包括标题内容。\n"
                    + "模板中的变量内容解释：\n"
                    + "变量1是给review打分，分数区间为0~100分。\n"
                    + "变量2 是code review发现的问题点，包括：可能的性能瓶颈、逻辑缺陷、潜在问题、安全风险、命名规范、注释、以及代码结构、异常情况、边界条件、资源的分配与释放等等\n"
                    + "变量3是具体的优化修改建议。\n"
                    + "变量4是你给出的修改后的代码。 \n"
                    + "变量5是代码中的优点。\n"
                    + "变量6是代码的逻辑和目的，识别其在特定上下文中的作用和限制\n"
                    + "\n"
                    + "必须要求：\n"
                    + "1. 以精炼的语言、严厉的语气指出存在的问题。\n"
                    + "2. 你的反馈内容必须使用严谨的markdown格式\n"
                    + "3. 不要携带变量内容解释信息。\n"
                    + "4. 有清晰的标题结构\n"
                    + "返回格式严格如下：\n"
                    + "# OpenAi 代码评审.\n"
                    + "### 😀代码评分：{变量1}\n"
                    + "#### 😀代码逻辑与目的：\n"
                    + "{变量6}\n"
                    + "#### ✅代码优点：\n"
                    + "{变量5}\n"
                    + "#### 🤔问题点：\n"
                    + "{变量2}\n"
                    + "#### 🎯修改建议：\n"
                    + "{变量3}\n"
                    + "#### 💻修改后的代码：\n"
                    + "{变量4}\n"
                    + "`;代码如下:";

    private static final String REVIEW_INSTRUCTION =
            "请根据以下diff内容进行代码审查：";

    @Override
    public List<ChatCompletionRequestDTO.Prompt> buildPrompts(String diffCode) {
        List<ChatCompletionRequestDTO.Prompt> prompts = new ArrayList<>();
        prompts.add(new ChatCompletionRequestDTO.Prompt("user", USER_INSTRUCTION));
        prompts.add(new ChatCompletionRequestDTO.Prompt("user", REVIEW_INSTRUCTION + diffCode));
        return prompts;
    }
}

// OpenAiCodeReviewService.java
@Override
protected String codeReview(String diffCode) throws Exception {
    logger.info("[trace] starting llm call");
    ChatCompletionRequestDTO chatCompletionRequest = new ChatCompletionRequestDTO();
    chatCompletionRequest.setModel(Model.GLM_4_FLASH.getCode());
    chatCompletionRequest.setMessages(promptProvider.buildPrompts(diffCode));

    try {
        ChatCompletionSyncResponseDTO completions = openAI.completions(chatCompletionRequest);
        logger.info("[trace] llm call done");
        ChatCompletionSyncResponseDTO.Message message = completions.getChoices().get(0).getMessage();
        return message.getContent();
    } catch (Exception e) {
        logger.error("[error] failed to call llm", e);
        throw new RuntimeException("Failed to call OpenAI model", e);
    }
}
```