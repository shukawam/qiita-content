---
title: Cohere を一通り試してみた
tags:
  - oci
  - Cohere
private: false
updated_at: '2023-08-22T23:29:52+09:00'
id: a8febfce15d92a0131b1
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

2 か月ほど前にこんな Press Release がでました。

https://www.oracle.com/jp/news/announcement/oracle-to-deliver-powerful-and-secure-generative-ai-service-for-business-2023-06-13/

エンタープライズ向けの AI プラットフォームを提供する Cohere と連携し、OCI 上で新たに Generative AI Service を提供する予定とのことです。
これは最低限、Cohere のことを知っておかなければならない！ということで学習がてら本記事では現在 Cohere から提供されている機能を [OCI Data Science](https://www.oracle.com/jp/artificial-intelligence/data-science/)[^1] を使って一通り試してみることにします。

尚、無料利用で発行できる Trial Key には Rate Limit や合計実行回数が厳し目に設定されていますので、ご注意ください。

https://docs.cohere.com/docs/going-live

[^1]: 機械学習のためのマネージド・プラットフォームを提供するサービス。本記事では、Jupyter Notebook の Oracle Managed な環境を利用します。

# Cohere から提供されるモデル・機能

[https://cohere.com/](https://cohere.com/) を眺めてみると、モデルが 2 つ提供されていることが分かります。

- Command
  - テキスト生成のモデルを提供
  - ユーザーのコマンドに従うように学習されている
- Embeddings
  - Embedding[^2] のモデルを提供
  - 多言語対応

[^2]: 自然言語を計算可能な形（多くの場合は、ベクトル空間）に変換すること

また、提供される機能は一部の Waitlist に登録しなければ使えないものを除き [API Reference](https://docs.cohere.com/reference/about) を見ると記載があります。

| endpoint         | overview                                                                                                   |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| /generate        | 与えられた入力を条件とし、リアルなテキストを生成しそれを返却する                                           |
| /embed           | テキストの Embedding を行い、浮動小数点のリストを返却する                                                  |
| /classify        | 入力されたテキストにどのラベルが合うのかを判定し、結果を返却する                                           |
| /tokenize        | 入力されたテキストをトークンに分割（形態素解析）し、その結果を返却する                                     |
| /detokenize      | BPE(byte-pair encoding)されたトークンを受け取り、そのテキスト表現を返却する                                |
| /detect-language | 入力されたテキストがどの言語で書かれたものかを特定し、その結果を返却する                                   |
| /summarize       | 入力されたテキストに対して、英語で要約を作成しその結果を返却する                                           |
| /rerank          | クエリとテキストのリストを受け取り、各テキストに関連スコアが割り当てられた順序付きの配列を生成し、返却する |

それでは早速試していきます！

## Common

非同期用のクライアントもありますが、今回は単純に機能を試すだけなので同期用のクライアントを使います。

```py
import cohere
api_key = '<your-api-key>'

co = cohere.Client(api_key)
co.check_api_key()
```

## Co.Generate

必要最低限のパラメータだけ指定して、実行してみます。

```py
res_generate = co.generate(
    prompt='Hello, how'
)

print(res_generate)
```

実行結果

```py
[cohere.Generation {
  id: 905bc1fa-cb5a-4082-b1d8-2704d426956b
  prompt: Hello, how
  text:  How can I help you?

  likelihood: None
  finish_reason: None
  token_likelihoods: None
}]
```

_How can I help you?_ というテキストが生成されて返ってきました。

これ以外にも、モデルを選択可能なパラメータ（model）や生成されるテキストの数やテキスト内に含まれる最大のトークン数などを指定するパラメータもあります。いくつか試した例を以下に記載します。

例: Nightly 版を使用し、最大 50 トークン含むテキストを 3 つ生成する

```py
res_generate = co.generate(
    prompt='Hello, how',
    model='command-nightly',
    num_generations=3,
    max_tokens=50
)

print(res_generate)
```

実行結果

```py
[cohere.Generation {
  id: 8eca3fb0-9a9b-4df2-997d-e5fba22ebfbb
  prompt: Hello, how
  text:  can I help you?
  likelihood: None
  finish_reason: None
  token_likelihoods: None
}, cohere.Generation {
  id: 2b5d7382-64cb-48a8-8bb0-33c574565def
  prompt: Hello, how
  text:  Hello! I'm a chatbot trained to be helpful and harmless. How can I assist you today?
  likelihood: None
  finish_reason: None
  token_likelihoods: None
}, cohere.Generation {
  id: 060903ac-241f-4a9f-bd46-32de661a0799
  prompt: Hello, how
  text:  are you doing today?
  likelihood: None
  finish_reason: None
  token_likelihoods: None
}]
```

例: 最も可能性の高い上位 5 個(k=5)のトークンのみを生成対象として考慮し、生成におけるランダム性を最大(temperature=5.0)にする

```py
res_generate = co.generate(
    prompt='Hello, how',
    model='command-nightly',
    num_generations=3,
    max_tokens=50,
    k=5,
    temperature=5.0
)

print(res_generate)
```

実行結果

```py
[cohere.Generation {
  id: 55f7482c-70ef-4846-be70-34af13df268a
  prompt: Hello, how
  text:  are you ?
  likelihood: None
  finish_reason: None
  token_likelihoods: None
}, cohere.Generation {
  id: d9286b01-bd12-4563-839b-6cf867f69873
  prompt: Hello, how
  text:  I’d like to thank the user.  Is this how they would prefer me to address them?  If so I can change that to their username.
How may I be of service?  If I can help with any tasks I
  likelihood: None
  finish_reason: None
  token_likelihoods: None
}, cohere.Generation {
  id: 6a2472a8-2663-4b83-a988-f164d1254707
  prompt: Hello, how
  text:  can I help you with something, or provide any assistance?

  likelihood: None
  finish_reason: None
  token_likelihoods: None
}]
```

例: 生成結果を JSON stream で受け取る（レスポンスの内容を少しずつレンダリングする UI にとって有益らしい）

```py
res_generate_streaming = co.generate(
    prompt='Hello, how',
    stream=True
)

for index, token in enumerate(res_generate_streaming):
    print(f"{index}: {token}")

res_generate_streaming.texts
```

実行結果

```py
0: StreamingText(index=0, text=' Hello', is_finished=False)
1: StreamingText(index=0, text='!', is_finished=False)
2: StreamingText(index=0, text=' How', is_finished=False)
3: StreamingText(index=0, text=' can', is_finished=False)
4: StreamingText(index=0, text=' I', is_finished=False)
5: StreamingText(index=0, text=' help', is_finished=False)
6: StreamingText(index=0, text=' you', is_finished=False)
7: StreamingText(index=0, text=' today', is_finished=False)
8: StreamingText(index=0, text='?', is_finished=False)
[' Hello! How can I help you today?']
```

その他詳細は、[API Reference - Co.Generate](https://docs.cohere.com/reference/generate) をご参照ください。

## Co.Embed

必要最低限のパラメータだけ指定して、実行してみます。

```py
res_embed = co.embed(
    texts=['hello', 'world']
)

print(res_embed)
```

実行結果（一部割愛してます）

```py
cohere.Embeddings {
  embeddings: [[1.6142578, 0.24841309, 0.5385742, -1.6630859, -0.27783203, 0.35888672, 1.3378906, -1.8261719, 0.89404297, 1.0791016, 1.0566406, 1.0664062, 0.20983887, ... , -2.4160156, 0.22875977, -0.21594238]]
  compressed_embeddings: []
  meta: {'api_version': {'version': '1'}}
}
```

Embedding された結果が返ってきました。また、Embedding の機能では、使用するモデルをパラメータで指定できるみたいです。

例: 軽量版モデル（model=embed-english-light-v2.0）を使ってみる

```py
res_embed = co.embed(
    texts=['hello', 'world'],
    model='embed-english-light-v2.0'
)

print(res_embed)
```

実行結果（一部割愛してます）

```py
cohere.Embeddings {
  embeddings: [[-0.16577148, -1.2109375, 0.54003906, -1.7148438, -1.5869141, 0.60839844, 0.6328125, 1.3974609, -0.49658203, -0.73046875, -1.796875, 1.5410156, 0.66064453, 0.9448242, -0.53515625, 0.24914551, 0.53222656, 0.23425293, 0.52685547, -1.3935547, 0.04095459, 0.8569336, -0.5620117, -0.42211914, 0.55371094, 3.5820312, 7.2890625, -1.2539062, 1.3583984, 0.12988281, -1.1660156, 0.124816895, ... ,0.87060547, 1.0205078, 0.5854492, -2.734375, -0.066589355, 1.8349609, 0.16430664, -0.26220703, -1.0625]]
  compressed_embeddings: []
  meta: {'api_version': {'version': '1'}}
}
```

例: multilingual 対応のモデル（model=embed-multilingual-v2.0）を使ってみる

```py
res_embed = co.embed(
    texts=['こんにちは', '世界'],
    model='embed-multilingual-v2.0'
)

print(res_embed)
```

実行結果（一部割愛してます）

```py
cohere.Embeddings {
  embeddings: [[0.2590332, 0.41308594, 0.24279785, 0.30371094, 0.04647827, 0.1361084, 0.41357422, -0.40063477, 0.2553711, 0.17749023, -0.1899414, -0.041900635, 0.20141602, 0.43017578, -0.5878906, 0.18054199, 0.42333984, 0.010749817, -0.56640625, 0.1517334, 0.14282227, 0.36767578, 0.26953125, 0.1418457, 0.28051758, 0.1661377, -0.13293457, 0.23620605, 0.08703613, 0.36914062, 0.22180176, ... ,0.027786255, -0.18530273, -0.24414062, 0.123168945, 0.6425781, 0.08831787, -0.21862793, -0.18237305, -0.031341553]]
  compressed_embeddings: []
  meta: {'api_version': {'version': '1'}}
}
```

その他の詳細は、[API Reference - Co.Embed](https://docs.cohere.com/reference/embed) をご参照ください。

## Co.Classify

必要最低限のパラメータだけ指定して、実行してみます。

```py
from cohere.responses.classify import Example

examples=[
  Example("Dermatologists don't like her!", "Spam"),
  Example("'Hello, open to this?'", "Spam"),
  Example("I need help please wire me $1000 right now", "Spam"),
  Example("Nice to know you ;)", "Spam"),
  Example("Please help me?", "Spam"),
  Example("Your parcel will be delivered today", "Not spam"),
  Example("Review changes to our Terms and Conditions", "Not spam"),
  Example("Weekly sync notes", "Not spam"),
  Example("'Re: Follow up from today's meeting'", "Not spam"),
  Example("Pre-read for tomorrow", "Not spam"),
]

inputs=[
  "Confirm your email address",
  "hey i need u to send some $",
]

res_classify = co.classify(
    inputs=inputs,
    examples=examples,
)

print(res_classify)
```

実行結果

```py
[Classification<prediction: "Not spam", confidence: 0.5661598, labels: {'Not spam': LabelPrediction(confidence=0.5661598), 'Spam': LabelPrediction(confidence=0.43384025)}>, Classification<prediction: "Spam", confidence: 0.9909811, labels: {'Not spam': LabelPrediction(confidence=0.009018883), 'Spam': LabelPrediction(confidence=0.9909811)}>]
```

_Confirm your email address_ は、スパムの場合もあればそうではない場合もあるので、`Not spam: 0.56..., Spam: 0.43...` は妥当な気がします。また、_hey i need u to send some $_ は、かなりスパムっぽいですが、結果もその通り（`Spam: 0.99, Not spam: 0.009`）でした。

Classification の機能では、他にモデルを指定するパラメータや truncate（最大トークン長より長い入力をどのように扱うか）が指定できます。

例: 軽量版モデル（model=embed-english-light-v2.0）で最大トークン長より長い入力を渡した際は、エラーが返却される（truncate=NONE）ようにする

```py
inputs_max_exceeded=[
  "Confirm your email address",
  "hey i need u to send some $",
  "Please sign the receipt attached for the arrival of new office facilities.",
  "Attached is the paper concerning with the cancellation of your current credit card.",
  "The monthly financial statement is attached within the email.",
  "The audit report you inquired is attached in the mail. Please review and transfer it to the related department.",
  "There is a message for you from 01478719696, on 2016/08/23 18:26:30. You might want to check it when you get a chance.Thanks!",
  "Dear John, I am attaching the mortgage documents relating to your department. They need to be signed in urgent manner.",
  "This email/fax transmission is confidential and intended solely for the person or organisation to whom it is addressed. If you are not the intended recipient, you must not copy, distribute or disseminate the information, or take any action in reliance of it. Any views expressed in this message are those of the individual sender, except where the sender specifically states them to be the views of any organisation or employer. If you have received this message in error, do not open any attachment but please notify the sender (above) deleting this message from your system. For email transmissions please rely on your own virus check no responsibility is taken by the sender for any damage rising out of any bug or virus infection.",
  "Sent with Genius Scan for iOS. http://bit.ly/download-genius-scan",
  "Hey John, as you requested, attached is the paycheck for your next month?s salary in advance.",
  "I am sending you the flight tickets for your business conference abroad next month. Please see the attached and note the date and time.",
  "Attached is the bank transactions made from the company during last month. Please file these transactions into financial record.",
  "example.com SV9100 InMail",
  "Attached file is scanned image in PDF format. Use Acrobat(R)Reader(R) or Adobe(R)Reader(R) of Adobe Systems Incorporated to view the document. Adobe(R)Reader(R) can be downloaded from the following URL: Adobe, the Adobe logo, Acrobat, the Adobe PDF logo, and Reader are registered trademarks or trademarks of Adobe Systems Incorporated in the United States and other countries.",
  "Here is the travel expense sheet for your upcoming company field trip. Please write down the approximate costs in the attachment.",
  "Attached is the list of old office facilities that need to be replaced. Please copy the list into the purchase order form.",
  "We are sending you the credit card receipt from yesterday. Please match the card number and amount.",
  "Queries to: <scanner@example.com",
  "Please find our invoice attached.",
  "Hello! We are looking for employees working remotely. My name is Margaret, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $3000-$6000. If you are interested in this offer, please visit",
  "Please see the attached copy of the report.",
  "Attached is a pdf file containing items that have shipped. Please contact us if there are any questions or further assistance we can provide",
  "Please see the attached license and send it to the head office.",
  "Please find attached documents as requested.",
  "We are looking for employees working remotely. My name is Enid, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $2000-$5400. If you are interested in this offer, please visit",
  "Your item #123456789 has been sent to you by carrier. He will arrive to youon 23th of September, 2016 at noon.",
  "Dear John, we are very sorry to inform you that the item you requested is out of stock. Here is the list of items similar to the ones you requested. Please take a look and let us know if you would like to substitute with any of them.",
  "Thank you for your order! Please find your final sales receipt attached to this email. Your USPS Tracking Number is: 123456789. This order will ship tomorrow and you should be able to begin tracking tomorrow evening after it is picked up. If you have any questions or experience any problems, please let us know so we can assist you. Thanks again and enjoy! Thanks,",
  "Thanks for using electronic billing. Please find your document attached",
  "Dear John, I attached the clients’ accounts for your next operation. Please look through them and collect their data. I expect to hear from you soon.",
  "You are receiving this email because the company has assigned you as part of the approval team. Please review the attached proposal form and make your approval decision. If you have any problem regarding the submission, please contact Gonzalo.",
  "this is to inform you that your Debit Card is temporarily blocked as there were unknown transactions made today. We attached the scan of transactions. Please confirm whether you made these transactions.",
  "The letter is attached. Please review his concerns carefully and reply him as soon as possible. King regards,",
  "Confirm your email address",
  "hey i need u to send some $",
  "Please sign the receipt attached for the arrival of new office facilities.",
  "Attached is the paper concerning with the cancellation of your current credit card.",
  "The monthly financial statement is attached within the email.",
  "The audit report you inquired is attached in the mail. Please review and transfer it to the related department.",
  "There is a message for you from 01478719696, on 2016/08/23 18:26:30. You might want to check it when you get a chance.Thanks!",
  "Dear John, I am attaching the mortgage documents relating to your department. They need to be signed in urgent manner.",
  "This email/fax transmission is confidential and intended solely for the person or organisation to whom it is addressed. If you are not the intended recipient, you must not copy, distribute or disseminate the information, or take any action in reliance of it. Any views expressed in this message are those of the individual sender, except where the sender specifically states them to be the views of any organisation or employer. If you have received this message in error, do not open any attachment but please notify the sender (above) deleting this message from your system. For email transmissions please rely on your own virus check no responsibility is taken by the sender for any damage rising out of any bug or virus infection.",
  "Sent with Genius Scan for iOS. http://bit.ly/download-genius-scan",
  "Hey John, as you requested, attached is the paycheck for your next month?s salary in advance.",
  "I am sending you the flight tickets for your business conference abroad next month. Please see the attached and note the date and time.",
  "Attached is the bank transactions made from the company during last month. Please file these transactions into financial record.",
  "example.com SV9100 InMail",
  "Attached file is scanned image in PDF format. Use Acrobat(R)Reader(R) or Adobe(R)Reader(R) of Adobe Systems Incorporated to view the document. Adobe(R)Reader(R) can be downloaded from the following URL: Adobe, the Adobe logo, Acrobat, the Adobe PDF logo, and Reader are registered trademarks or trademarks of Adobe Systems Incorporated in the United States and other countries.",
  "Here is the travel expense sheet for your upcoming company field trip. Please write down the approximate costs in the attachment.",
  "Attached is the list of old office facilities that need to be replaced. Please copy the list into the purchase order form.",
  "We are sending you the credit card receipt from yesterday. Please match the card number and amount.",
  "Queries to: <scanner@example.com",
  "Please find our invoice attached.",
  "Hello! We are looking for employees working remotely. My name is Margaret, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $3000-$6000. If you are interested in this offer, please visit",
  "Please see the attached copy of the report.",
  "Attached is a pdf file containing items that have shipped. Please contact us if there are any questions or further assistance we can provide",
  "Please see the attached license and send it to the head office.",
  "Please find attached documents as requested.",
  "We are looking for employees working remotely. My name is Enid, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $2000-$5400. If you are interested in this offer, please visit",
  "Your item #123456789 has been sent to you by carrier. He will arrive to youon 23th of September, 2016 at noon.",
  "Dear John, we are very sorry to inform you that the item you requested is out of stock. Here is the list of items similar to the ones you requested. Please take a look and let us know if you would like to substitute with any of them.",
  "Thank you for your order! Please find your final sales receipt attached to this email. Your USPS Tracking Number is: 123456789. This order will ship tomorrow and you should be able to begin tracking tomorrow evening after it is picked up. If you have any questions or experience any problems, please let us know so we can assist you. Thanks again and enjoy! Thanks,",
  "Thanks for using electronic billing. Please find your document attached",
  "Dear John, I attached the clients’ accounts for your next operation. Please look through them and collect their data. I expect to hear from you soon.",
  "You are receiving this email because the company has assigned you as part of the approval team. Please review the attached proposal form and make your approval decision. If you have any problem regarding the submission, please contact Gonzalo.",
  "this is to inform you that your Debit Card is temporarily blocked as there were unknown transactions made today. We attached the scan of transactions. Please confirm whether you made these transactions.",
  "The letter is attached. Please review his concerns carefully and reply him as soon as possible. King regards,",
  "Confirm your email address",
  "hey i need u to send some $",
  "Please sign the receipt attached for the arrival of new office facilities.",
  "Attached is the paper concerning with the cancellation of your current credit card.",
  "The monthly financial statement is attached within the email.",
  "The audit report you inquired is attached in the mail. Please review and transfer it to the related department.",
  "There is a message for you from 01478719696, on 2016/08/23 18:26:30. You might want to check it when you get a chance.Thanks!",
  "Dear John, I am attaching the mortgage documents relating to your department. They need to be signed in urgent manner.",
  "This email/fax transmission is confidential and intended solely for the person or organisation to whom it is addressed. If you are not the intended recipient, you must not copy, distribute or disseminate the information, or take any action in reliance of it. Any views expressed in this message are those of the individual sender, except where the sender specifically states them to be the views of any organisation or employer. If you have received this message in error, do not open any attachment but please notify the sender (above) deleting this message from your system. For email transmissions please rely on your own virus check no responsibility is taken by the sender for any damage rising out of any bug or virus infection.",
  "Sent with Genius Scan for iOS. http://bit.ly/download-genius-scan",
  "Hey John, as you requested, attached is the paycheck for your next month?s salary in advance.",
  "I am sending you the flight tickets for your business conference abroad next month. Please see the attached and note the date and time.",
  "Attached is the bank transactions made from the company during last month. Please file these transactions into financial record.",
  "example.com SV9100 InMail",
  "Attached file is scanned image in PDF format. Use Acrobat(R)Reader(R) or Adobe(R)Reader(R) of Adobe Systems Incorporated to view the document. Adobe(R)Reader(R) can be downloaded from the following URL: Adobe, the Adobe logo, Acrobat, the Adobe PDF logo, and Reader are registered trademarks or trademarks of Adobe Systems Incorporated in the United States and other countries.",
  "Here is the travel expense sheet for your upcoming company field trip. Please write down the approximate costs in the attachment.",
  "Attached is the list of old office facilities that need to be replaced. Please copy the list into the purchase order form.",
  "We are sending you the credit card receipt from yesterday. Please match the card number and amount.",
  "Queries to: <scanner@example.com",
  "Please find our invoice attached.",
  "Hello! We are looking for employees working remotely. My name is Margaret, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $3000-$6000. If you are interested in this offer, please visit",
  "Please see the attached copy of the report.",
  "Attached is a pdf file containing items that have shipped. Please contact us if there are any questions or further assistance we can provide",
  "Please see the attached license and send it to the head office.",
  "Please find attached documents as requested.",
  "We are looking for employees working remotely. My name is Enid, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $2000-$5400. If you are interested in this offer, please visit",
  "Your item #123456789 has been sent to you by carrier. He will arrive to youon 23th of September, 2016 at noon.",
  "Dear John, we are very sorry to inform you that the item you requested is out of stock. Here is the list of items similar to the ones you requested. Please take a look and let us know if you would like to substitute with any of them.",
  "Thank you for your order! Please find your final sales receipt attached to this email. Your USPS Tracking Number is: 123456789. This order will ship tomorrow and you should be able to begin tracking tomorrow evening after it is picked up. If you have any questions or experience any problems, please let us know so we can assist you. Thanks again and enjoy! Thanks,",
  "Thanks for using electronic billing. Please find your document attached",
  "Dear John, I attached the clients’ accounts for your next operation. Please look through them and collect their data. I expect to hear from you soon.",
  "You are receiving this email because the company has assigned you as part of the approval team. Please review the attached proposal form and make your approval decision. If you have any problem regarding the submission, please contact Gonzalo.",
  "this is to inform you that your Debit Card is temporarily blocked as there were unknown transactions made today. We attached the scan of transactions. Please confirm whether you made these transactions.",
  "The letter is attached. Please review his concerns carefully and reply him as soon as possible. King regards,",
  "Confirm your email address",
  "hey i need u to send some $",
  "Please sign the receipt attached for the arrival of new office facilities.",
  "Attached is the paper concerning with the cancellation of your current credit card.",
  "The monthly financial statement is attached within the email.",
  "The audit report you inquired is attached in the mail. Please review and transfer it to the related department.",
  "There is a message for you from 01478719696, on 2016/08/23 18:26:30. You might want to check it when you get a chance.Thanks!",
  "Dear John, I am attaching the mortgage documents relating to your department. They need to be signed in urgent manner.",
  "This email/fax transmission is confidential and intended solely for the person or organisation to whom it is addressed. If you are not the intended recipient, you must not copy, distribute or disseminate the information, or take any action in reliance of it. Any views expressed in this message are those of the individual sender, except where the sender specifically states them to be the views of any organisation or employer. If you have received this message in error, do not open any attachment but please notify the sender (above) deleting this message from your system. For email transmissions please rely on your own virus check no responsibility is taken by the sender for any damage rising out of any bug or virus infection.",
  "Sent with Genius Scan for iOS. http://bit.ly/download-genius-scan",
  "Hey John, as you requested, attached is the paycheck for your next month?s salary in advance.",
  "I am sending you the flight tickets for your business conference abroad next month. Please see the attached and note the date and time.",
  "Attached is the bank transactions made from the company during last month. Please file these transactions into financial record.",
  "example.com SV9100 InMail",
  "Attached file is scanned image in PDF format. Use Acrobat(R)Reader(R) or Adobe(R)Reader(R) of Adobe Systems Incorporated to view the document. Adobe(R)Reader(R) can be downloaded from the following URL: Adobe, the Adobe logo, Acrobat, the Adobe PDF logo, and Reader are registered trademarks or trademarks of Adobe Systems Incorporated in the United States and other countries.",
  "Here is the travel expense sheet for your upcoming company field trip. Please write down the approximate costs in the attachment.",
  "Attached is the list of old office facilities that need to be replaced. Please copy the list into the purchase order form.",
  "We are sending you the credit card receipt from yesterday. Please match the card number and amount.",
  "Queries to: <scanner@example.com",
  "Please find our invoice attached.",
  "Hello! We are looking for employees working remotely. My name is Margaret, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $3000-$6000. If you are interested in this offer, please visit",
  "Please see the attached copy of the report.",
  "Attached is a pdf file containing items that have shipped. Please contact us if there are any questions or further assistance we can provide",
  "Please see the attached license and send it to the head office.",
  "Please find attached documents as requested.",
  "We are looking for employees working remotely. My name is Enid, I am the personnel manager of a large International company. Most of the work you can do from home, that is, at a distance. Salary is $2000-$5400. If you are interested in this offer, please visit",
  "Your item #123456789 has been sent to you by carrier. He will arrive to youon 23th of September, 2016 at noon.",
  "Dear John, we are very sorry to inform you that the item you requested is out of stock. Here is the list of items similar to the ones you requested. Please take a look and let us know if you would like to substitute with any of them.",
  "Thank you for your order! Please find your final sales receipt attached to this email. Your USPS Tracking Number is: 123456789. This order will ship tomorrow and you should be able to begin tracking tomorrow evening after it is picked up. If you have any questions or experience any problems, please let us know so we can assist you. Thanks again and enjoy! Thanks,",
  "Thanks for using electronic billing. Please find your document attached",
  "Dear John, I attached the clients’ accounts for your next operation. Please look through them and collect their data. I expect to hear from you soon.",
  "You are receiving this email because the company has assigned you as part of the approval team. Please review the attached proposal form and make your approval decision. If you have any problem regarding the submission, please contact Gonzalo.",
  "this is to inform you that your Debit Card is temporarily blocked as there were unknown transactions made today. We attached the scan of transactions. Please confirm whether you made these transactions.",
  "The letter is attached. Please review his concerns carefully and reply him as soon as possible. King regards,",
]

res_classify = co.classify(
    inputs=inputs_max_exceeded,
    examples=examples,
    model='embed-english-light-v2.0',
    truncate='END'
)

res_classify.classifications
```

実行結果

```py
---------------------------------------------------------------------------
CohereAPIError                            Traceback (most recent call last)

# ... omit

CohereAPIError: invalid request: inputs cannot contain more than 96 elements, received 136
```

最大 96 個の入力までしか渡せない inputs に 136 の入力を与えたところちゃんとエラーで返ってきました。

その他の詳細は、[API Reference - Co.Classify](https://docs.cohere.com/reference/classify) をご参照ください。

## Co.Tokenize

必要最低限のパラメータだけ指定して、実行してみます。

```py
res_tokenize = co.tokenize(
    text='tokenize me !D',
)

print(res_tokenize)
```

実行結果

```py
cohere.Tokens {
  tokens: [10002, 2261, 2012, 3362, 43]
  token_strings: ['token', 'ize', ' me', ' !', 'D']
  meta: {'api_version': {'version': '1'}}
}
```

この他に、モデルを選択するパラメータはあるのですが、`command` 以外何が選択可能なのか [API Reference - Co.Tokenize](https://docs.cohere.com/reference/tokenize) を参照しても良く分かりませんでした。

## Co.Detokenize

必要最低限のパラメータだけ指定して、実行してみます。

```py
tokens = [10002, 2261, 2012, 3362, 43] # tokenize me !D

res_detokenize = co.detokenize(
    tokens=tokens
)

print(res_detokenize)
```

実行結果

```py
tokenize me !D
```

この他に、モデルを選択するパラメータはあるのですが、`command` 以外何が選択可能なのか [API Reference - Co.Detokenize](https://docs.cohere.com/reference/detokenize-1) を参照しても良く分かりませんでした。

## Co.Detect_language

必要最低限のパラメータだけ指定して、実行してみます。

```py
res_detect_language = co.detect_language(
    texts=['Hello world', "'Здравствуй, Мир'", 'こんにちは世界', '世界你好', '안녕하세요']
)

res_detect_language
```

実行結果

```py
cohere.DetectLanguageResponse {
  results: [Language<language_code: "en", language_name: "English">, Language<language_code: "ru", language_name: "Russian">, Language<language_code: "ja", language_name: "Japanese">, Language<language_code: "zh", language_name: "Chinese">, Language<language_code: "ko", language_name: "Korean">]
  meta: {'api_version': {'version': '1'}}
}
```

ちゃんと検出されていそうです。Detect_language はこれ以外に指定可能なパラメータはありませんでした。

## Co.Summarize

必要最低限のパラメータだけ指定して、実行してみます。

```py
text = (
  "Ice cream is a sweetened frozen food typically eaten as a snack or dessert. "
  "It may be made from milk or cream and is flavoured with a sweetener, "
  "either sugar or an alternative, and a spice, such as cocoa or vanilla, "
  "or with fruit such as strawberries or peaches. "
  "It can also be made by whisking a flavored cream base and liquid nitrogen together. "
  "Food coloring is sometimes added, in addition to stabilizers. "
  "The mixture is cooled below the freezing point of water and stirred to incorporate air spaces "
  "and to prevent detectable ice crystals from forming. The result is a smooth, "
  "semi-solid foam that is solid at very low temperatures (below 2 °C or 35 °F). "
  "It becomes more malleable as its temperature increases.\n\n"
  "The meaning of the name \"ice cream\" varies from one country to another. "
  "In some countries, such as the United States, \"ice cream\" applies only to a specific variety, "
  "and most governments regulate the commercial use of the various terms according to the "
  "relative quantities of the main ingredients, notably the amount of cream. "
  "Products that do not meet the criteria to be called ice cream are sometimes labelled "
  "\"frozen dairy dessert\" instead. In other countries, such as Italy and Argentina, "
  "one word is used fo\r all variants. Analogues made from dairy alternatives, "
  "such as goat's or sheep's milk, or milk substitutes "
  "(e.g., soy, cashew, coconut, almond milk or tofu), are available for those who are "
  "lactose intolerant, allergic to dairy protein or vegan."
)

co_summarize = co.summarize(
    text=text
)

print(co_summarize)
```

実行結果

```py
SummarizeResponse(id='5d566208-06f3-4da0-ac54-fd590c22777f', summary='Ice cream is a frozen food, usually eaten as a snack or dessert. It is made from milk or cream, and is flavored with a sweetener and a spice or fruit. It can also be made by whisking a flavored cream base and liquid nitrogen together. Food coloring and stabilizers may be added. The mixture is cooled and stirred to incorporate air spaces and prevent ice crystals from forming. The result is a smooth, semi-solid foam that is solid at low temperatures. It becomes more malleable as its temperature increases. The meaning of the name "ice cream" varies from one country to another. Some countries, such as the US, have specific regulations for the commercial use of the term "ice cream". Products that do not meet the criteria to be called ice cream are sometimes labeled "frozen dairy dessert". Dairy alternatives, such as goat\'s or sheep\'s milk, or milk substitutes, are available for those who are lactose intolerant or allergic to dairy protein.', meta={'api_version': {'version': '1'}})
```

> Ice cream is a frozen food, usually eaten as a snack or dessert. It is made from milk or cream, and is flavored with a sweetener and a spice or fruit. It can also be made by whisking a flavored cream base and liquid nitrogen together. Food coloring and stabilizers may be added. The mixture is cooled and stirred to incorporate air spaces and prevent ice crystals from forming. The result is a smooth, semi-solid foam that is solid at low temperatures. It becomes more malleable as its temperature increases. The meaning of the name "ice cream" varies from one country to another. Some countries, such as the US, have specific regulations for the commercial use of the term "ice cream". Products that do not meet the criteria to be called ice cream are sometimes labeled "frozen dairy dessert". Dairy alternatives, such as goat\'s or sheep\'s milk, or milk substitutes, are available for those who are lactose intolerant or allergic to dairy protein.

ちゃんと要約されてそうです。これ以外にも出力される要約の長さ（length）が指定できたり、出力形式（format）や要約の際に元の文章からどれだけ抽出するか？（extractiveness）などといったパラメータが指定できるようです。いくつか試した例を以下に記載します。

例: 長さを短く（length=short）し、箇条書き形式（format=bullets）で出力する

```py
co_summarize = co.summarize(
    text=text,
    length='short',
    format='bullets'
)

print(co_summarize)
```

実行結果

```py
SummarizeResponse(id='fefca5fe-1dd4-4fff-91b0-622fc9fc64ec', summary='- Ice cream is a frozen dessert made from milk or cream.\n- It is flavored with a sweetener and a spice or fruit.\n- It can also be made with dairy alternatives.', meta={'api_version': {'version': '1'}})
```

※見やすくするために改行を加えています。

> - Ice cream is a frozen dessert made from milk or cream.
> - It is flavored with a sweetener and a spice or fruit.
> - It can also be made with dairy alternatives.

例: 軽量版のモデルを使い（model=command-light）、元の文章からの抽出度を高くし（extractiveness=high）、箇条書き形式で出力する

```py
co_summarize = co.summarize(
    text=text,
    model='command-light',
    length='short',
    format='bullets',
    extractiveness='high'
)

print(co_summarize)
```

実行結果

```py
SummarizeResponse(id='df22373b-dd85-4b8f-a6c4-0aac6db32898', summary='- Ice cream is a sweetened frozen food typically eaten as a snack or dessert.\n- It may be made from milk or cream and is flavored with a sweetener, either sugar or an alternative.\n- It can also be made by whisking a flavored cream base and liquid nitrogen together.\n- The result is a smooth, semi-solid foam that is solid at very low temperatures.\n- It becomes more malleable as its temperature increases.\n- The meaning of the name varies from one country to another.', meta={'api_version': {'version': '1'}})
```

※見やすくするために改行を加えています。

> - Ice cream is a sweetened frozen food typically eaten as a snack or dessert.
> - It may be made from milk or cream and is flavored with a sweetener, either sugar or an alternative.
> - It can also be made by whisking a flavored cream base and liquid nitrogen together.
> - The result is a smooth, semi-solid foam that is solid at very low temperatures.
> - It becomes more malleable as its temperature increases.
> - The meaning of the name varies from one country to another.

確かに、元の文章からの抽出度が高くなりました（e.g._Ice cream is a sweetened frozen food typically eaten as a snack or dessert._）

その他詳細は、[API Reference - Co.Summarize](https://docs.cohere.com/reference/summarize-2) をご参照ください。

## Co.Rerank

必要最低限のパラメータだけ指定して、実行してみます。

```py
docs = [
    'Carson City is the capital city of the American state of Nevada.',
    'The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan.',
    'Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district.',
    'Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states.'
]

co_rerank = co.rerank(
    model='rerank-english-v2.0',
    query='What is the capital of the United States?',
    documents=docs
)

print(co_rerank)
```

[API Reference - Co.Rerank](https://docs.cohere.com/reference/rerank-1) によると、model のパラメータは required ではないのですが、付与せずに実行したところ `TypeError: rerank() missing 1 required positional argument: 'model'` とエラーログが出力されたので必須のパラメータだと思われます。ご注意を。

実行結果

```bash
[RerankResult<document['text']: Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district., index: 2, relevance_score: 0.98005307>, RerankResult<document['text']: Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states., index: 3, relevance_score: 0.27904198>, RerankResult<document['text']: Carson City is the capital city of the American state of Nevada., index: 0, relevance_score: 0.10194652>, RerankResult<document['text']: The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan., index: 1, relevance_score: 0.0721122>]
```

> Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district.

順序付き配列もきちんと関連度順に並んでそうでした。

これ以外にも multilingual なモデルが指定できたり（model=rerank-multilingual-v2.0）、順序付き配列のうちレスポンス含める数を指定（top_n=3）するためのパラメータが指定できるようです。いくつか試した例を以下に記載します。

例: 日本語を試してみる（model=rerank-multilingual-v2.0）

先ほどの docs, query を DeepL にかけた結果を用いてみます。

```py
docs_ja = [
    'カーソンシティはアメリカ・ネバダ州の州都である',
    '北マリアナ諸島連邦は太平洋に浮かぶ島々である。首都はサイパンである。',
    ' ワシントンD.C.（Washington, D.C.）は、アメリカ合衆国の首都である。連邦区である',
    '死刑制度はアメリカ合衆国が国である以前から存在する。2017年現在、死刑は50州のうち30州で合法である。'
]

co_rerank_ja = co.rerank(
    model='rerank-multilingual-v2.0',
    query='アメリカの首都は何ですか？',
    documents=docs_ja
)

print(co_rerank_ja)
```

出力結果

```py
[RerankResult<document['text']:  ワシントンD.C.（Washington, D.C.）は、アメリカ合衆国の首都である。連邦区である, index: 2, relevance_score: 0.9998813>, RerankResult<document['text']: カーソンシティはアメリカ・ネバダ州の州都である, index: 0, relevance_score: 0.37903354>, RerankResult<document['text']: 北マリアナ諸島連邦は太平洋に浮かぶ島々である。首都はサイパンである。, index: 1, relevance_score: 0.23899394>, RerankResult<document['text']: 死刑制度はアメリカ合衆国が国である以前から存在する。2017年現在、死刑は50州のうち30州で合法である。, index: 3, relevance_score: 0.0007014083>]
```

> ワシントン D.C.（Washington, D.C.）は、アメリカ合衆国の首都である。連邦区である

デフォルトのモデル同様に日本語でも順序付き配列はきちんと関連度順に並んでそうでした。

例: 日本語（model=rerank-multilingual-v2.0）で最も相関の高いもののみ（top_n=1）を返却する。

```py
docs_ja = [
    'カーソンシティはアメリカ・ネバダ州の州都である',
    '北マリアナ諸島連邦は太平洋に浮かぶ島々である。首都はサイパンである。',
    ' ワシントンD.C.（Washington, D.C.）は、アメリカ合衆国の首都である。連邦区である',
    '死刑制度はアメリカ合衆国が国である以前から存在する。2017年現在、死刑は50州のうち30州で合法である。'
]

co_rerank_ja = co.rerank(
    model='rerank-multilingual-v2.0',
    query='アメリカの首都は何ですか？',
    documents=docs_ja,
    top_n=1,
)

print(co_rerank_ja)
```

実行結果

```py
[RerankResult<document['text']:  ワシントンD.C.（Washington, D.C.）は、アメリカ合衆国の首都である。連邦区である, index: 2, relevance_score: 0.9998813>]
```

return_documents（返却される配列にdocumentを含めるか否か）というパラメータも試したのですが、こちらはうまく動作しませんでした。

その他詳細は、[API Reference - Co.Rerank](https://docs.cohere.com/reference/rerank-1) をご参照ください。

# おわりに

今回、色々実験した Notebook はこちらから参照できます。

https://github.com/shukawam/notebooks/blob/main/cohere/getting-started-cohere.ipynb
