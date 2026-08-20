# Step 13 — まとめ・付録

⬅️ 前へ: [Step 12 — トラブルバッファ](step12-troubleshoot.md) ｜ 🏠 [目次](README.md)

---

## まとめ（本日の振り返り）
本日は、Microsoft Foundry ポータル上で以下を体験しました。

1. **素のモデル**の限界を確認（最新情報・社内情報を持たない）
2. **単一 Agent** を作成し、instructions と **Web Search**（最新情報）で回答
3. 事前準備済みの **MCP ツール** を接続して外部連携を体験
4. 事前準備済みの **Foundry IQ ナレッジ** を接続し、社内資料を根拠にした**根拠付き回答**
5. **トレース**で内部処理を確認（運用・デバッグの入口）
6. **手動評価**で回答品質を可視化
7. **（発展・任意）Microsoft 365 / Teams への公開**で、普段使うツールから Agent を呼べる体験
8. **（発展・任意）Hosted Agent デプロイ**でコード資産をマネージド ホストへ公開
9. **クリーンアップ**でリソースを整理

知識源が「素のモデル → Web Search → Foundry IQ」と広がり、さらに「トレースで説明し、評価で改善する」という運用の流れを一気通貫で体験しました。

## 受講者の成果物
- [ ] instructions を設定した単一 Agent
- [ ] Web Search / Foundry IQ を使った根拠付き回答（正常例）
- [ ] 手動評価シート（正常例・失敗例・引用有無）
- [ ] トレース閲覧結果 1件以上
- [ ] クリーンアップ確認表

## 次の一歩（発展の方向性）
- 複数ツール・複数 Agent の連携（マルチエージェント）
- 自動評価（Evaluation）による品質の継続的モニタリング
- MCP サーバー / Foundry IQ ナレッジの**自作**（本日は接続のみ。構築は別セッションで）

---

## 付録 A: 前提・留意事項
- 本ハンズオンは **ポータル UI 中心**。UI は更新されることがあるため、メニュー名が異なる場合は近い名称を探す。
- **MCP サーバー**と **Foundry IQ ナレッジ**は**事前準備済みの共有リソース**。本日は「接続して使う」のみ。構築手順は別途案内。
- 使用しない技術: **Assistants API**（非推奨。2026-08-26 に提供終了予定のため本ハンズオンでは扱わない）。
  - 出典: <https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/assistants>

## 付録 B: 事実 / 推測の区別について
本教材は、一次情報で確認できた事項（**事実**）と、UI 変更等により差異が生じうる事項（**推測**）を区別しています。手順中のメニュー名・トグル位置などは環境により異なる場合があるため、迷ったら講師に確認してください。

## 付録 C: 主要参照元
- Microsoft Foundry ポータル: <https://ai.azure.com/>
- Foundry Agent Service GA: <https://devblogs.microsoft.com/foundry/foundry-agent-service-ga/>
- Web Search ツール: <https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/web-search>
- Observability / Tracing: <https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/observability>
- Assistants API（非推奨）: <https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/assistants>

---

⬅️ 前へ: [Step 12 — トラブルバッファ](step12-troubleshoot.md) ｜ 🏠 [目次](README.md)
