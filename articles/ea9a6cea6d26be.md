---
title: "KarabinerでEnter長推しをaltに設定する"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["karabiner"]
published: true
---

```json
 {
   "description": "Held Enter to alt",
   "manipulators": [
       {
           "from": {
               "key_code": "return_or_enter",
               "modifiers": { "optional": ["any"] }
           },
           "parameters": { "basic.to_if_held_down_threshold_milliseconds": 1000 },
           "to": [{ "key_code": "left_alt" }],
           "to_if_alone": [{ "key_code": "return_or_enter" }],
           "type": "basic"
       }
   ]
 }
 ```
