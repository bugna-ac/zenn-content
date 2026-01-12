# Roo固有というより、設計とプロンプト discipline の問題が強い　は、どういう意味？もう少し解説して。
この記事に出てくる「問題」のかなりの部分は、Rooというツール固有のバグというより、「人間側の設計とプロンプトのやり方がまずいと、どのLLMエージェントでも同じことが起こる」という意味合いが強い、という指摘。この記事の具体例に即して噛み砕きます。[1][2]

***

## メリット（この見方の意図）

- 広域 `try...except` の乱発や700〜2000行の巨大ファイル化は、「エラーを消せ」「これ全部直して」と丸投げするプロンプトと、もともと大きすぎる設計があると、Rooに限らず多くのLLMエージェントがやらかすパターン。[1][3]
- コンテキスト上限やモデルごとの差は、Rooだけでなく「LLM＋エージェント」という構造ならどこでも出る制約で、その制約を前提に設計・粒度・指示を調整すれば、他ツールに乗り換えても再利用しやすい考え方になる。[1][2]

***

## デメリット（この見方の限界）

- Roo固有の設定UI、コンテキスト制御の癖、モデルの組み合わせ方など、実際にはツール固有の挙動もあるので、「全部ユーザのdisciplineの問題」と片付けると、Roo側の改善余地を見落とす。[4][5][6]
- ユーザ設計・プロンプトをどれだけ整えても、Rooの実装上のバグ（コンテキストサイズ計算ミスなど）があれば、やはりRoo固有のトラブルとして残る。[7][5]

***

## 想定リスク（放置した場合に起きること）

- 「Rooが悪い」とだけ認識して設計・プロンプトを見直さないと、他のエージェント（Cursor/Cline/社内エージェントなど）に乗り換えても、同じように広域except地獄・巨大ファイル・行数ベース分割が再発する。[1][3]
- 逆にRoo前提の運用ルールを作り込みすぎると、「Rooではうまく回るが、別エージェントではフィットしない」組織文化になり、将来のツール変更コストが跳ね上がるリスクがある。[1][8]

***

## 盲点・反対意見（もう一歩突っ込んだ見方）

- 設計・プロンプト discipline を強調しすぎると、「人間がツールに適応しろ」という一方向の最適化になり、ツール側でやるべきガードレール（例外飲み込みの抑制、過大diffの警告、責務ベース分割の支援）を要求しなくなるという反論はあり得る。[3][9]
- 一部の高性能エージェントは、自己修正ループや静的解析を組み合わせて、広域exceptなどを自動で避ける設計も研究されているため、「disciplineで殴るしかない」という前提も将来的には古くなる可能性がある。[9][3]

情報源
[1] 6bc9775186eaf7 https://zenn.dev/bugnabuna/articles/6bc9775186eaf7
[2] Working with Large Projects | Roo Code Documentation https://docs.roocode.com/advanced-usage/large-projects
[3] Dignified Python: 10 Rules to Improve your LLM Agents - Dagster https://dagster.io/blog/dignified-python-10-rules-to-improve-your-llm-agents
[4] Roo Code: A Guide With 7 Practical Examples https://www.datacamp.com/tutorial/roo-code
[5] Blown context window with long lines · Issue #5775 https://github.com/RooCodeInc/Roo-Code/issues/5775
[6] Controlling Context Length : r/RooCode https://www.reddit.com/r/RooCode/comments/1kcjcuf/controlling_context_length/
[7] Roo Code ignores the `Context Window Size` setting for ... https://github.com/RooCodeInc/Roo-Code/issues/4475
[8] Roo Code vs Cline: Best AI Coding Agents for VS Code (2026) - Qodowww.qodo.ai › blog › roo-code-vs-cline https://www.qodo.ai/blog/roo-code-vs-cline/
[9] Self-correcting Code Generation Using Multi-Step Agent https://deepsense.ai/resource/self-correcting-code-generation-using-multi-step-agent/
[10] Roo Code – The AI dev team that gets things done https://roocode.com
[11] Roo Code gives you a whole dev team of AI agents ... https://github.com/RooCodeInc/Roo-Code
[12] Roo Code is AMAZING - AI VSCode Extension (better than Cursor?) https://www.youtube.com/watch?v=r5T3h0BOiWw
[13] Roo Code - Visual Studio Marketplace https://marketplace.visualstudio.com/items?itemName=RooVeterinaryInc.roo-cline
[14] Your Ultimate AI Coding Agent: Roo Code + Visual Studio Code 🚀 https://www.youtube.com/watch?v=hRxjMTyB-GA
[15] Installing Roo Code | Roo Code Documentation https://docs.roocode.com/getting-started/installing
[16] Seeker: Enhancing Exception Handling in Code with a LLM-based ... https://arxiv.org/html/2410.06949v1
