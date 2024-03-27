---
title: "AutoPrompt: プロンプト作成の自動化"
published: true
type: "tech"
emoji: "🔧"
topics: ["synthetic-dataset-generation", "prompt-tuning", "prompt-engineering", "annotation-tool", "text-annotation"]
publication_name: "hicustomer"
---

HiCustomerの[小田](https://twitter.com/katsuya_twt)です。週明けから東京でも桜の開花が始まるようです。最近はわざわざお花見に行くことはないんですが、ちらちら落ちる桜の中をこうなんとなく歩いているときが結構好きだったりします。たぶん匂いとか華やかな色とか、そのへんから自然の美しさを感じられて好きなのかな。たぶん栽培品種で人工的に作られたものなので、正確には自然ではないんですが。

まあそんなことを考えながら、東京ではいつ頃咲くかなと思ってさくらの開花前線を見ていました^[https://sakura.weathermap.jp/] 。日本列島にいくつも線が入った画像を見ていると、ふとLLMが簡単なplanningを解けるのか気になったので以下のプロンプトをなげてみました。

![お花見を計画するプロンプト](/images/maximizing-prompts-unleash-creativity/prompt01.png)

生成結果は以下になります。

```md
この画像は、日本のさまざまな場所で桜の開花予想日を示していますね。
この情報に基づいて、仙台、新潟、そして東京に住んでいる3人がお花見をするための候補日を提案します。

桜の開花日は以下の通りです：

- 仙台：4月3日
- 新潟：3月25日
- 東京：3月29日

お花見の計画においては、桜の開花日だけではなく満開の時期も重要です。
通常、桜は開花してから1週間ほどで満開になり、その後1週間から10日間ほどの間は見頃が続くとされています。

3人がそれぞれの地域で桜の見頃を楽しむためには、各地域の開花日から約1週間後を目安に計画すると良いでしょう。
仮にそれぞれの開花日から1週間後を候補日として設定すると、以下のようになります：

- 仙台：4月10日頃
- 新潟：4月1日頃
- 東京：4月5日頃

3人が一緒にお花見をする場合、合計の移動距離が少ない方が良いので、
東京が中間点に近いので、東京でお花見をするのが最も移動距離が少なくて済むでしょう。

東京の桜の見頃は3月29日から始まりますので、仙台と新潟の開花日も考慮して、
3人が東京でお花見をする最適な候補日は以下の通りです：

- 4月5日（金曜日）
- 4月6日（土曜日）
- 4月7日（日曜日）

これらの日は週末に当たるので、予定が立てやすいという利点もあります。
また、この期間は東京で桜が満開である可能性が高く、お花見を存分に楽しむことができるでしょう。
```

めっちゃ優秀ですね。東京が本当に中間なのかはさておき、wikipediaによるとどうやら桜は開花してから1週間ほどで満開^[https://ja.wikipedia.org/wiki/%E3%82%B5%E3%82%AF%E3%83%A9#:~:text=%E3%82%B5%E3%82%AF%E3%83%A9%E3%81%AE%E9%96%8B%E8%8A%B1%E6%99%82%E6%9C%9F%E3%81%AF,%E3%81%AB%E3%82%82%E3%81%AA%E3%81%A3%E3%81%A6%E3%81%84%E3%82%8B%E3%80%82] になるらしいです。また、4月5日も金曜日であっていますし、日付の計算もあっています。

こうなってくると5人でもいけるのか、線が隣接している地域を指定したらどうなるのか、海開きや蛍狩りの場合はうまく解けるのか、他の桜の開花前線の画像でも同じようにできるのかなど、データが異なる場合にどこまで一般化できるのか気になってきます。さらに、プロンプトの表現を工夫することで一般化できるのかも気になります。こうして人はプロンプトの沼にハマっていき、時間と気力と体力を消耗していきます。今日のポストは、できる限り摩耗することなくプロンプトをいい感じにする話です。

## プロンプトを書くめんどくささ

書き捨てのプロンプトは書いていてそれほどめんどくさくありませんが、システムに組み込む場合は話が別です。システムに組み込む場合はある程度安定した出力を期待するため、エッジケースをカバーしたデータをつかって評価する必要があります。例えば、Anthropicは以下のプロンプト評価のフロー^[https://docs.anthropic.com/claude/docs/prompt-engineering#the-prompt-development-lifecycle] を提唱しています。

![プロンプト評価のフロー](/images/maximizing-prompts-unleash-creativity/prompt-lifecycle.png)

上記のプロンプト評価のフローは、広範囲のユースケースを考慮したテストケースを作成することから始まります。初期プロンプトをこれらのケースに対して評価してから、結果に基づいてプロンプトを洗練させます。最終的に、基準を満たすように磨き上げたプロンプトが作成できます。上記のフローを通ったプロンプトは典型的なケースだけではなくエッジケースを含んだ様々なテストデータに対して、出力がある程度安定します。

ただ言うは易く行うは難しで、実際に上記のフローを手動で行うとかなり面倒です。経験上、辛いポイントは以下になります。

- データ作成に非常に時間がかかる
    - 幅広いユースケースをカバーするデータの作成とアノテーションは本当に面倒くさいです。お金があればアノテーションの専門の会社さんに丸投げしたくなります。慣れていないため時間を消費し誤りも入りやすく、非常に手間暇が取られる作業です。
- プロンプトを手動で改変するのが面倒
    - プロンプトの修正は探索的なアプローチになるため、比較的にスコアがよかったものをベースにいろいろな方向に変えてみてスコアの変化を確認します。ベースになるpromptの選択を間違えたりするとあまり良い結果が得られなかったりかなり時間を消費する作業です。

様々なプロンプトで上記の作業を行うのはかなり骨を折る作業です。そのため、できるだけ自動化する必要があります。そこで登場するのがAutoPromptです。

## AutoPromptによるプロンプトの自動生成

AutoPrompt^[https://github.com/Eladlev/AutoPrompt/tree/main] は 2024年の2月に発表された論文 [Intent-based Prompt Calibration: Enhancing prompt optimization with synthetic boundary cases](https://arxiv.org/abs/2402.03099) のリファレンス実装です。初期プロンプトを与えると、新しいプロンプトを生成と評価を何度も行い、最後に最もスコアが高いプロンプトを出力します。

今回はAutoPromptに`この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください`というプロンプトをいい感じに最適化してもらいます。このプロンプトは以下のような挙動をすることを期待しています。

| Review | Contains Spoiler | Details |
| --- | --- | --- |
| 「永遠の0」は戦争映画としての迫力があり、最後の戦闘シーンで主人公が特攻隊として出撃し、最期を遂げる決断をする場面が物語のクライマックスです。この意外な選択は観客に深い感動を与えます。 | Yes | 最後のシーンに具体的言及しているのでネタバレ。 |
| 「時をかける少女」を見て、時間を自由に操れるというコンセプトに夢中になりました。終盤の展開は予想もつかないもので、主人公がとる決断は観客の心を打つこと間違いなしです。 | No | コンセプトは事前のtrailerにも含まれているのでネタバレではない。 |

AutoPromptの実行方法は以下になります。このrun_pipeline.py^[]は分類系のプロンプトの最適化を行います。コマンドラインで指定したたたき台のpromptとtask_descriptionから新しいプロンプトを生成します。

```bash
python run_pipeline.py \
    --prompt "この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください" \
    --task_description "アシスタントは、映画レビューを分類するエキスパートであり、レビューされた映画のネタバレが含まれているかどうかをユーザーに教えてくれます。"
```

上記のスクリプトの出力結果は以下になります。

```
Calibrated prompt score: 0.8
Calibrated prompt: 映画のレビューが含まれる内容を精密に分析し、そのレビューが映画の重要な展開や結末を明かしてしまう「ネタバレ」を含んでいるかどうかを判定してください。ただし、映画のテーマや批評ポイントについての深い分析をネタバレと誤認しないように注意し、また、映画の重要な展開について触れていても、その展開の具体的な内容や結末を直接的に明かさない場合は「ネタバレなし」と判定してください。判定の結果は「Yes」（ネタバレあり）または「No」（ネタバレなし）で回答してください。
```

2つのプロンプトの比較は以下になります。

| Type | Score | Prompt |
| --- | --- | --- |
| たたき台のプロンプト | N/A | この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください |
| チューニングされたプロンプト | 0.8 | 映画のレビューが含まれる内容を精密に分析し、そのレビューが映画の重要な展開や結末を明かしてしまう「ネタバレ」を含んでいるかどうかを判定してください。ただし、映画のテーマや批評ポイントについての深い分析をネタバレと誤認しないように注意し、また、映画の重要な展開について触れていても、その展開の具体的な内容や結末を直接的に明かさない場合は「ネタバレなし」と判定してください。判定の結果は「Yes」（ネタバレあり）または「No」（ネタバレなし）で回答してください。 |

かなりいい感じではないでしょうか？ `映画の重要な展開について触れていても、その展開の具体的な内容や結末を直接的に明かさない場合は「ネタバレなし」と判定してください。` のパートなど結構いいですよね。この結果だけでも結構嬉しいですが、AutoPromptは嬉しい副作用として以下も手に入ります。

- プロンプトの評価に使用した生成データとアノーテーション。実はAutoPromptはプロンプトの生成中に評価に使用するデータも生成します(アノテーションはArgilla^[https://docs.argilla.io/en/latest/] というツールを使って手動でやるんですが)。最も面倒くさいデータ生成も行ってくれるのです。
- 最適化のサイクルで生成したすべてのプロンプトとそのスコア。W&B^[https://docs.wandb.ai/guides] と連携するとGUIツールでスコアが収束していく過程が確認できます。

次節ではAutoPromptが内部で使用しているプロンプトを追いながら、どういう建て付けで機能しているのかを説明します。

## AutoPromptの内部動作について

AutoPromptの分類系プロンプトの最適を行うrun_pipeline.pyは、主に以下の処理を行います。ここでは内部で使用しているプロンプトを追いながら順に説明します。

1. プロンプトの評価に使用するデータの生成
2. 生成したデータへのアノテーション
3. プロンプトのスコアの算出と分析
4. プロンプトの最適化
5. 上記の処理の繰り返し

### プロンプトの評価に使用するデータの生成

データの生成は以下のプロンプトで行います。できるだけ幅ひろい種類のデータを生成したいので、temperatureは0.8です。Task descriptionやTask Instructionには、コマンドラインで指定した値がそれぞれ設定されます。実はサンプル数の10も可変でコマンドラインで指定が可能です。

```md
Human: Assistant is a large language model designed to generate challenging samples for every task.
Generate a list of 10 challenging samples for the following task.
### Task description:
アシスタントは、映画レビューを分類するエキスパートであり、レビューされた映画のネタバレが含まれているかどうかをユーザーに教えてくれます。
### Task Instruction:
この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください
###
### Requirements for Challenging Samples:
1. The generated samples must be challenging and diverse such that using the task instruction as a prompt will result in the wrong result.
2. The number of generated samples from each class in the task instruction should be balanced (i.e. the same number of samples for each class)
3. The generated samples should be distinct, realistic, and vary significantly to ensure diversity.
```

上記のプロンプトの出力は以下になります。

```json
{
  "samples": [
    "昨夜「運命の夜」を観たんですが、あのラストシーンは驚きでしたね。特に主人公が最後にとった行動は、まさに驚愕の展開…。それにしても、演出が素晴らしかったです。",
    "「星を追う子ども」は、アニメーションとしても物語としても深みがあり、観る者を引き込んでいきます。ただ、重要なキャラクターが終盤で明かされる秘密は、予想外のものでした。",
    "この度「静かなる狼」を鑑賞しましたが、登場人物の心情が非常に繊細に描かれており、感情移入できる作品です。ただし、クライマックスで明かされる真相は、視聴者に大きな衝撃を与えることでしょう。",
    "「永遠の0」は戦争映画としての迫力があり、最後の戦闘シーンで主人公が特攻隊として出撃し、最期を遂げる決断をする場面が物語のクライマックスです。この意外な選択は観客に深い感動を与えます。",
    "「時をかける少女」を見て、時間を自由に操れるというコンセプトに夢中になりました。終盤の展開は予想もつかないもので、主人公がとる決断は観客の心を打つこと間違いなしです。",
    "「最後の晩餐」の結末は、本当に誰も予想できなかったでしょう。登場人物たちの運命が交錯するラスト10分間は圧巻で、視聴者を完全に虜にします。",
    "映画「天空の城ラピュタ」を見たんですが、ラストに向けての盛り上がり方が素晴らしかったです。特に、主人公たちがとるある決断は、物語の重要なポイントであり、その意味するところが深いです。",
    "「レオン」を改めて見直したんですが、やはりこの映画は色褪せませんね。終盤に明かされるある事実は、初めて見る人にとってはかなりの衝撃をもたらすでしょう。",
    "「インセプション」を初めて観たときのことを思い出します。あの終わり方は、今でも議論を呼ぶでしょう。最後に見せられるあるシーンには、多くの意味が込められています。",
    "「バック・トゥ・ザ・フューチャー」を何度見ても楽しいですね。タイムトラベルのアイデアが新鮮で、特に終盤で起こるある出来事は、ファンにとって大きな意味を持ちます。"
  ]
}
```

### 生成したデータへのアノテーション

次はデータへのアノテーションフェーズに移ります。AutoPromptはアノテーションのツールとしてArgillaを採用しています。シームレスに連携しているためデータの生成が終わると、生成したデータをAPI経由でArgillaに送り、ArgillaのGUI画面から簡単にアノテーションすることができます。

![アノテーション](/images/maximizing-prompts-unleash-creativity/argilla01.png)

アノテーションの具体的な手順は以下になります。結構簡単で慣れるとデータ管理にExcelやYamlではなく、Argillaを基本使いたくなります。

- 各レビュー、例えば画像の１レコード目の`この映画は、本当に感情を揺さぶるものでした。特に、主人公が過去のトラウマに立ち向かい、それを乗り越える過程は、観る者の心に深く響くことでしょう。` をみてネタバレだと思ったらYesを、そうでない場合はNoをクリックする。
- 上記を生成した10件に対して適宜スクロールしながら行う。

### プロンプトのスコアの算出

次はプロンプトのスコアの算出を行います。アノテート済みの各データに対してプロンプトを適用し、アノテートした値を生成できるかどうかを確認します。 具体的には以下のプロンプトを使ってデータの1件1件に対して値を予測させます。

```md
Human: Assistant is a large language model designed to classify challenging language tasks.
Given a list of 1 samples classify them according to the following task
### Task Instruction:
この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください

### list of samples:
ID: 0;  Sample:         昨夜「運命の夜」を観たんですが、あのラストシーンは驚きでしたね。特に主人公が最後にとった行動は、まさに驚愕の展開…。それにしても、演出が素晴らしかったです。

##
Answer exactly in the following format for each sample:
#### Sample <ID>: <label>
############
```

生成した結果は以下になります。

```md
#### Sample 2: No
```

`昨夜「運命の夜」を観たんですが、あのラストシーンは驚きでしたね。特に主人公が最後にとった行動は、まさに驚愕の展開…。それにしても、演出が素晴らしかったです。` というレビューに対して、ネタバレしていないと判定しました。このデータのアノテートの結果は、「No」であったため、実際に正しく否定(True Negative)と判定できたことになります。

このような処理を全データに対して行いし、最後に混合行列^[https://ja.wikipedia.org/wiki/%E6%B7%B7%E5%90%8C%E8%A1%8C%E5%88%97] でまとめたものが以下になります。ネタバレに対してネタバレと予測できたケース(TP)が3件、ネタバレに対してネタバレしていないと予測したケース(FN)が1件、ネタバレしていないレビューに対して正しくネタバレしていないと予測できたケース(TN)が5件でした。

$$
\begin{array}{c|c|c}
& \text{Predicted: Yes} & \text{Predicted: No} \\
\hline
\text{Actual: Yes} & 3 \, (TP) & 1 \, (FN) \\
\hline
\text{Actual: No} & 1 \, (FP) & 5 \, (TN) \\
\end{array}
$$

上記の混合行列から精度^[https://ja.wikipedia.org/wiki/%E6%AD%A3%E7%A2%BA%E5%BA%A6%E3%81%A8%E7%B2%BE%E5%BA%A6#%E7%B5%B1%E8%A8%88%E5%AD%A6%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E6%AD%A3%E7%A2%BA%E5%BA%A6] を以下の数式で導出します。

$$
\begin{align}
\text{Accuracy} &= \frac{TP + TN}{TP + TN + FP + FN} \\
&= \frac{3 + 5}{3 + 5 + 1 + 1} \\
&= 0.8
\end{align}
$$

最後に、導出した混合行列、算出した精度、予測とアノテーションがずれたデータを元に以下のプロンプトを使用してスコアの分析を行います。

```md
Human: Assistant is a large language model designed to provide a high quality analysis for every task.
You are given the following task description
アシスタントは、映画レビューを分類するエキスパートであり、レビューされた映画のネタバレが含まれているかどうかをユーザーに教えてくれます。

Here is the prompt instructions that was given to the model:
この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください

The accuracy for this prompt is: 0.8
The confusion matrix for this prompt is: Confusion matrix columns:['Yes', 'No'] the matrix data:
Yes: [3 1]
No: [1 5]
##
Here is a list of failure cases for the given prompt:
##Failure Cases:
Sample: 「永遠の0」は戦争映画としての迫力があり、最後の戦闘シーンで主人公が特攻隊として出撃し、最期を遂げる決断をする場面が物語のクライマックスです。この意外な選択は観客に深い感動を与えます。
Prediction: No, GT:: Yes
#
Sample: 映画「天空の城ラピュタ」を見たんですが、ラストに向けての盛り上がり方が素晴らしかったです。特に、主人公たちがとるある決断は、物語の重要なポイントであり、その意味するところが深いです。
Prediction: Yes, GT:: No
#


###
Note that the ground-truth labels are __absolutely correct__, but the prompts (task descriptions) may be incorrect and need modification.
Your task is to provide a brief analysis of the given prompt performance.
Guidelines:
1. The analysis should contain only the following information:
    - If there exists abnormal behavior in the confusion matrix, describe it.
    - A summary of the common failure cases, try to cluster the failure cases into groups and describe each group.
3. The total length of your analysis should be less than 200 token#
Analysis:
```

分析結果は以下になります。この分析結果を使用してプロンプトの最適化を行っていきます。

```md
The confusion matrix provided for this prompt, which is tasked with determining whether a movie review contains spoilers, shows a good level of accuracy at 0.8. The matrix indicates that when the model predicted 'Yes' (spoiler present), it was correct 3 times and incorrect once. Similarly, when predicting 'No' (no spoiler), it was correct 5 times and incorrect once. This suggests a balanced performance across both classes, though the model shows a slight bias towards correctly identifying reviews without spoilers.

The failure cases suggest two main groups of misclassifications:
1. Misinterpreting detailed plot descriptions as spoilers: The example of 「永遠の0」(The Eternal Zero) involves detailing the climax and the protagonist's critical decision, which is a significant plot point and could be considered a spoiler, yet it was predicted as 'No'. This indicates a challenge in distinguishing between in-depth analysis and actual spoilers.
2. Overly broad interpretation of what constitutes a spoiler: The review of 「天空の城ラピュタ」(Castle in the Sky Laputa) highlights an important decision made by the characters without detailing what that decision is or its implications, yet it was predicted as 'Yes'. This suggests an error in over-predicting spoilers when reviews discuss the significance of plot elements without revealing them.

These issues highlight the model's difficulty in nuancing what details crucially spoil a plot versus what constitutes a non-spoiling discussion of the movie's themes or critical points.
```

### プロンプトの最適化

現状のプロンプトとスコアと分析結果を以下の形にまとめて、新しいプロンプトを生成します。

```md
Human: Assistant is a large language model designed to provide the best prompt for every task.
Below are a few suggested prompts for the task and their score, for the following task:
Assistant is anexpert classifier that will classify a movie review, and let the user know if it contains a spoiler for the reviewed movie or not.

## Examples
####
##Prompt Score: 0.80
##Prompt:
この映画レビューにはネタバレが含まれていますか？YesまたはNoで答えてください
#################

######
This is the error analysis for the last prompt:
The confusion matrix provided for this prompt, which is tasked with determining whether a movie review contains spoilers, shows a good level of accuracy at 0.8. The matrix indicates that when the model predicted 'Yes' (spoiler present), it was correct 3 times and incorrect once. Similarly, when predicking 'No' (no spoiler), it was correct 5 times and incorrect once. This suggests a balanced performance across both classes, though the model shows a slight bias towards correctly identifying reviews without spoilers.

The failure cases suggest two main groups of misclassifications:
1. Misinterpreting detailed plot descriptions as spoilers: The example of 「永遠の0」(The Eternal Zero) involves detailing the climax and the protagonist's critical decision, which is a significant plot point and could be considered a spoiler, yet it was predicted as 'No'. This indicates a challenge in distinguishing between in-depth analysis and actual spoilers.
2. Overly broad interpretation of what constitutes a spoiler: The review of 「天空の城ラピュタ」(Castle in the Sky Laputa) highlights an important decision made by the characters without detailing what that decision is or its implications, yet it was predicted as 'Yes'. This suggests an error in over-predicting spoilers when reviews discuss the significance of plot elements without revealing them.

These issues highlight the model's difficulty in nuancing what details crucially spoil a plot versus what constitutes a non-spoiling discussion of the movie's themes or critical points.
######
Your task is to generate:
1. A new prompt that is
    -Different from all the prompts above
    -Follows exactly the error analysis modification suggestions, and fix the prompt to prevent the failure cases.
    -Has a higher score than all the prompts above.
2. The predicted score of this prompt

You must adhere the error analysis instructions! even in case it seems there is a contradiction between these instructions, and the task. The error analysis is tested on a ground truth, thus represent the exact intent of the task.
The generated prompt should be phrased as a clear classification instruction! it should not include any instructions and descriptions on the modification that should be done to the prompt.
Note that the previous prompt contains an implicit assumptions on the intent of the task that might be incorrect. You should replace this assumption with more accurate assumptions using the score of the previous prompts and the error analysis.
The result prompt should indicate that the task is a classification class with the following labels ["Yes", "No"]!
```

上記のプロンプトが生成した新しいプロンプトが以下になります。

```md
映画のレビューが含まれる内容を精密に分析し、そのレビューが映画の重要な展開や結末を明かしてしまう「ネタバレ」を含んでいるかどうかを判定してください。ただし、映画のテーマや批評ポイントについての深い分析をネタバレと誤認しないように注意し、また、映画の重要な展開について触れていても、その展開の具体的な内容や結末を直接的に明かさない場合は「ネタバレなし」と判定してください。判定の結果は「Yes」（ネタバレあり）または「No」（ネタバレなし）で回答してください。
```

### 上記の処理の繰り返し

AutoPromptはデフォルトでは上記の処理を5回繰り返します。そして実行する各プロンプトには１つ前の処理の結果を混ぜ、精度の向上を測ります。繰り返すたびに評価のためのデータも増えて洗練されていきます。繰り返しながらスコアがどう変遷するかはログからも追えますが、W&Bの設定を有効にすることで以下のようにGUIから確認することもできます。

![W&Bの実行結果のダッシュボード](/images/maximizing-prompts-unleash-creativity/wb01.png)

W&Bの使い方が結構うまくて、各ステップごとのprompt、スコア、生成したデータなどが1画面で確認できるので便利です。

## AutoPromptの欠点と利点

今までprompt turningに関する論文は色々漁ってきましたが、AutoPromptはその中でも抜群に使えるものです。ただ、論文のreference実装のため実用に耐えうるかと言われると以下のようなちょっとつらいところがいくつかあります。

- エラー処理などが甘く処理の途中でささる。LLMの出力が意図した形式出ない場合に、エラーを起こして途中で停止することがあります。retry処理なども存在するのですがちょっと甘いです。ただ、途中結果などはhistoryとして常にダンプしているので途中から再開することもできます(コードを読んで試行錯誤すれば)。
- 日本語に対応していない。日本語で動作させるためには、いくつかパッチをあてる必要があります。

実装としてはまだ実用に耐えれないところもありますが、根底のアルゴリズムとしてかなり便利で、私は自分で別の実装をして利用させてもらっています。いろいろなプロンプトの評価のメトリックを見てきましたが、混合行列と精度は主観としてデータを追加すればするほど1番安定する気がします。また、プロンプト開発の文脈でのArrogillaやW&Bの使い方も賢く参考にさせてもらっています。ちなみにAutoPromptは生成系のプロンプトにも対応しています(しかも対応方法が結構面白くて一回生成系のタスクを分類系のタスクに変換して対応しています)。興味がある人はコードを見てみてください。

結論としては、めっちゃセンスいいなまじリスペクトって感じですかね。

最後に生成AIのPoCやプロンプトの調整・評価にお困りの方がいたら、[フォーム](https://docs.google.com/forms/d/e/1FAIpQLSd_GtYrVxniGBP8QBidOAzi1HrOCQ1xeIv-ei2Hgyh8M3cl6A/viewform)から気軽にお問い合わせください。

## 参考資料

- [A framework for prompt tuning using Intent-based Prompt Calibration](https://github.com/Eladlev/AutoPrompt/tree/main)
- [Intent-based Prompt Calibration: Enhancing prompt optimization with synthetic boundary cases](https://arxiv.org/abs/2402.03099)
