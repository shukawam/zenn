---
title: "Llama-3.2-90B-Vision-InstructをOCI Generative AI Serviceから試す"
emoji: "🦙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OCI"]
published: true
---

## はじめに

OCI(Oracle Cloud Infrastructure)の Generative AI Service といえば、Cohere 社の Command モデルというイメージが個人的には強かったですが、実はリリース当初から Llama も使用することができます。Cohere 社が提供しているモデルは埋め込みモデルを除き、現状マルチモーダルに対応できるものはありませんので、そのようなユースケースでは LLama を選択するパターンもあるかな？ということで、OCI SDK for Python から `Llama-3.2-90B-Vision-Instruct` を使う方法を解説します ✋

## OCI SDK で操作する

最小限のコード例は以下のようになります。

```py
import base64
from oci.auth.signers.instance_principals_security_token_signer import InstancePrincipalsSecurityTokenSigner
from oci.generative_ai_inference.generative_ai_inference_client import GenerativeAiInferenceClient
from oci.generative_ai_inference.models import (
    ImageContent,
    ImageUrl,
    TextContent,
    ChatDetails,
    OnDemandServingMode,
    GenericChatRequest,
    Message
)

compartment_id = "<your-compartment-id>"
image_path = "<your-image-path>"

# ... 1
client = GenerativeAiInferenceClient(
    config={},
    signer=InstancePrincipalsSecurityTokenSigner(),
    service_endpoint="https://inference.generativeai.ap-osaka-1.oci.oraclecloud.com"
)

def get_image_base64(image_path: str):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode()

def chat_with_image(base64_image_content: str) -> str:
    response = client.chat(
        chat_details=ChatDetails(
            compartment_id=compartment_id,
            serving_mode=OnDemandServingMode(
                model_id="meta.llama-3.2-90b-vision-instruct",
            ),
            # ... 2
            chat_request=GenericChatRequest(
                messages=[
                    Message(
                        role="SYSTEM",
                        content=[
                            TextContent(
                                text="Please explain about given images."
                            )
                        ]
                    ),
                    Message(
                        role="USER",
                        content=[ImageContent(
                            # ... 3
                            image_url=ImageUrl(
                                url=f"data:image/jpg;base64,{base64_image_content}"
                            )
                        )]
                    )
                ]
            )
        )
    )
    return response.data

def main():
    base64_image_content = get_image_base64(image_path)
    response = chat_with_image(base64_image_content)
    print(f"{response=}")


if __name__ == "__main__":
    main()
```

ポイントを簡単に解説します。

### `GenerativeAIInferenceClient` の初期化

今回、実装した Python プログラムは OCI Compute 上で実行しているので当該 Compute を含む動的グループに適切な IAM Policy(e.g. `allow dynamic-group <hoge> to use generative-ai-family in compartment <hoge>`) を書くことで Generative AI Service に対する認可制御を実施しています。

```py
client = GenerativeAiInferenceClient(
    config={},
    signer=InstancePrincipalsSecurityTokenSigner(),
    service_endpoint="https://inference.generativeai.ap-osaka-1.oci.oraclecloud.com"
)
```

IAM User ベースで制御（API Key を用いた制御）を行いたい場合は、以下のように書き直すと良いでしょう。

```py
config = from_file("~/.oci/config", "DEFAULT")
client = GenerativeAiInferenceClient(
    config=config,
    service_endpoint="https://inference.generativeai.ap-osaka-1.oci.oraclecloud.com"
)
```

### `chat_request` には、`GenericChatRequest` を用いる

`chat_request` には、[BaseChatRequest](https://docs.oracle.com/en-us/iaas/tools/python/2.151.0/api/generative_ai_inference/models/oci.generative_ai_inference.models.BaseChatRequest.html) のサブクラスである `GenericChatRequest` か `CohereChatRequest` を用いますが、今回は Llama を使用するため `GenericChatRequest` を用います。

```py
chat_request=GenericChatRequest(
    messages=[
        Message(
            # ... 省略 ...
        ),
        Message(
            # ... 省略 ...
        )
    ]
)
```

### `ImageUrl` の形式に注意する

`ImageUrl` で指定必須なのは、`url` パラメータだけですが、`url` パラメータの指定は一癖あります。[API ドキュメント - oci.generative_ai_inference.models.ImageUrl](https://docs.oracle.com/en-us/iaas/tools/python/2.151.0/api/generative_ai_inference/models/oci.generative_ai_inference.models.ImageUrl.html#oci.generative_ai_inference.models.ImageUrl) をみてみると以下のような記載があります。

```txt
[Required] Gets the url of this ImageUrl. The base64 encoded image data.

Example for a png image:
{
  “type”: “IMAGE”, “imageUrl”: {
    “url”: “data:image/png;base64,<base64 encoded image content>”
  }
}
```

ということで、`url` パラメータにはいくつかの Prefix と Base64 エンコードした画像ファイルを渡せば良いということがわかります。

```py
Message(
    role="USER",
    content=[ImageContent(
        # ... 3
        image_url=ImageUrl(
            url=f"data:image/jpg;base64,{base64_image_content}"
        )
    )]
)
```

### 実行してみた

上記の Python ファイルと同じディレクトリに以下の画像を置いて確認してみました。

![koinobori](/images/generative-ai-llama-3.2/koinobori.jpg)

```py
response={
  "chat_response": {
    "api_format": "GENERIC",
    "choices": [
      {
        "finish_reason": "stop",
        "index": 0,
        "logprobs": {
          "text_offset": null,
          "token_logprobs": null,
          "tokens": null,
          "top_logprobs": null
        },
        "message": {
          "content": [
            {
              "text": "The image depicts a scene featuring three large koi fish-shaped windsocks suspended from a tall pole against a blue sky with white clouds.\n\n**Key Features:**\n\n*   **Koi Fish Windsocks:** The windsocks are designed to resemble koi fish, a symbol of good luck and prosperity in Japanese culture. They are typically flown during the Japanese holiday of Children's Day (Kodomo no Hi) on May 5th.\n*   **Design and Colors:** Each windsock features a different design and color scheme, adding visual interest to the scene. The top windsock is white with a black tail and features Japanese characters in red, while the middle windsock is also white but has a pink body and features more Japanese characters. The bottom windsock is green and pink, with a more intricate design.\n*   **Suspension:** The windsocks are attached to a tall pole that appears to be made of metal or wood. The pole is adorned with a decorative top, which adds to the festive atmosphere of the scene.\n*   **Background:** The background of the image is a blue sky with white clouds, which provides a serene and peaceful contrast to the vibrant colors of the windsocks.\n\n**Cultural Significance:**\n\n*   **Children's Day:** The image is likely taken during the celebrations of Children's Day, a Japanese holiday that honors the health and happiness of children. The koi fish windsocks are a traditional symbol of this holiday, representing good luck and prosperity for children.\n*   **Japanese Culture:** The image showcases elements of Japanese culture, including the design and colors of the windsocks, which are inspired by traditional Japanese art and craftsmanship.\n\n**Overall Impression:**\n\nThe image conveys a sense of joy and celebration, with the colorful windsocks and festive atmosphere evoking feelings of happiness and optimism. The use of traditional Japanese symbols and designs adds a layer of cultural significance to the image, making it a unique and interesting representation of Japanese culture.",
              "type": "TEXT"
            }
          ],
          "name": null,
          "role": "ASSISTANT",
          "tool_calls": []
        }
      }
    ],
    "time_created": "2025-05-08T06:55:34.641000+00:00"
  },
  "model_id": "meta.llama-3.2-90b-vision-instruct",
  "model_version": "1.0.0"
}
```

と返却され、確かに画像に対する説明が適切にされていることがわかります。ちなみに、[Hugging Face の Llama-3.2-90B-Vision-Instruct の Model Card](https://huggingface.co/meta-llama/Llama-3.2-90B-Vision-Instruct) をみるとわかりますが、画像 + テキストの場合は、英語しかサポートしていないため日本語での応答がしたい場合は、別の多言語対応のモデル（例えば、Cohere など）を用いて入力と出力を翻訳するなどの工夫が必要でしょう。

## 終わりに

今回は、マルチモーダルに対応した Llama-3.2-90B-Vision-Instruct を OCI の Generative AI Service から試してみました ✋ 一部パラメータの指定方法に癖がありますので、同じようなところで悩んでいる方の一助になれば幸いです。

## 参考

https://huggingface.co/meta-llama/Llama-3.2-90B-Vision-Instruct

https://docs.oracle.com/en-us/iaas/tools/python/2.151.0/api/generative_ai_inference/models/oci.generative_ai_inference.models.ImageUrl.html#oci.generative_ai_inference.models.ImageUrl
