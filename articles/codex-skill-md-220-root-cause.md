---
title: "Codex が SKILL.md を 220 行で切る原因は、Codex 自身の prompt の 1 行だった"
emoji: "🔧"
type: "tech"
topics: ["codex", "claudecode", "ai", "agent", "llm"]
published: true
---

[前回の記事](https://zenn.dev/haru0416/articles/codex-skill-md-220-lines)で、Codex CLI が SKILL.md を 220 行で打ち切る現象を計測した。「model が学習データから引いてきた SKILL.md の見慣れた長さ」を原因と推測していたが、これは半分しか合っていなかった。

本当の原因は、**Codex CLI 自身の prompt が「Read only enough to follow the workflow」と書いている**ことだった。openai/codex に既に Issue が立ち、proposed patch まで用意されている。今もそのままになっている。

## 原因の 1 行

Codex CLI v0.128.0 の `codex-rs/core-skills/src/render.rs` を読むと、skill 起動時に model へ渡る prompt の中にこれが入っている。

```rust
// codex-rs/core-skills/src/render.rs
pub const SKILLS_HOW_TO_USE_WITH_ABSOLUTE_PATHS: &str = r###"
- How to use a skill (progressive disclosure):
  1) After deciding to use a skill, open its `SKILL.md`.
     Read only enough to follow the workflow.
  ...
"###;
```

**「Read only enough to follow the workflow」** ── これだけ。skill 本文を「partial にしか読むな」と明示している。私の Codex セッションログを `grep` すると、この文字列が会話にそのまま入っていた。runtime に届いている文字通りの指示だ。

model はその「enough」を、自分の prior で違う数字に解く。前回観測した cap のばらつきは、gpt-5.5 で **220**、gpt-5.4 で **260** ── それぞれの model の「enough」だった。

## 最新版で直っているか

最新 release は **v0.134.0 (2026-05-26 公開)**、私の計測時点から 6 versions 進んでいる。`rust-v0.128.0` と `rust-v0.134.0` で render.rs を diff すると、変更は `plugin_id: None,` の 1 行追加 (skill analytics 用) だけ。**prompt 文字列は 1 文字も変わっていない**。

念のため v0.134.0 で再計測したが、cap 分布も cap-test の二手目挙動も chain も、すべて旧と同じ。Issue #16479 が直されない限り、update では消えない。

## 既に Issue が立っていた

[openai/codex#16479](https://github.com/openai/codex/issues/16479) ── 2026-04-01 に hannesrudolph 氏が立てた issue。同じ問題を指摘していて、proposed wording も書かれている:

```diff
- 1) After deciding to use a skill, open its `SKILL.md`. Read only enough to follow the workflow.
+ 1) After deciding to use a skill, open and read its `SKILL.md` in full. ...
```

cluedesc 氏が patch まで作っている ([`cluedesc:fix/skills-read-skill-md-first`](https://github.com/cluedesc/codex/tree/fix/skills-read-skill-md-first))。ただし openai/codex は **invitation でしか PR を受け付けていない**ので、外部からは merge できない。Issue は OPEN のまま、4 コメントすべて外部、OpenAI のチームからは応答ゼロ。最終更新は 2026-04-06、**2 ヶ月近く動いていない**。

なお live main には同じ wording が 2 箇所 (`SKILLS_HOW_TO_USE_WITH_ABSOLUTE_PATHS` + `SKILLS_HOW_TO_USE_WITH_ALIASES`) ある。cluedesc patch は片方しか触っておらず、2026-04-24 の `#19098` で第 2 const が後から追加されたため。current main に rebase + 両方更新が要る。

## 私の実測が Issue の主張を裏付けるか

#16479 は wording を変えるべきだと議論の側から指摘している。runtime でそれが本当に起きていることを、計測で 3 角度から見せる。

### `~/.codex/sessions/` 全 209 セッションの cap 分布

71 セッションが SKILL.md を sed で読みにいき、計 143 個の read を出した。これを model と cap で並べると:

| model | cap=220 | cap=240 | cap=260 | その他 |
|---|---:|---:|---:|---:|
| gpt-5.5 | **80** (76%) | 10 | 11 | 4 (cap-test の段階読み) |
| gpt-5.4 | 9 | 2 | **27** (71%) | 0 |

**同じ prompt、同じ codex exec 経路、model だけ変えると分布が逆転する**。これは Issue が言う「the model has to guess what counts as 'enough'」を、そのまま実測で見せている。

### Harness を変えると cap が消える

通常 Claude Code は Claude に、Codex CLI は OpenAI のモデルに紐づいている。[claudex](https://github.com/EdamAme-x/claudex) でこの組み合わせをずらすと、Claude Code の harness のまま gpt-5.5 を動かせる ── 同じ model を Codex CLI の prompt に晒さない経路。

11 セッション回したところ、first move は `Skill(...)` tool 呼び出しで **11/11**、`sed -n '1,Np' .../SKILL.md` は **0/11**。Codex CLI の prompt が無い経路では、同 model でも cap は **そもそも発生しない**。

### Skill 自身に「もっと読め」と書けば、model は読む

「cap-test」という skill を用意した。Iron Law marker を line 220 を超えた位置に置き、prefix の中で「`sed -n '221,$p'` で残りを読め」と明示している。gpt-5.5 で **7/7**、gpt-5.4 で **4/4** が marker を取得 (= 二手目を打って 221+ を読んだ)。

つまり model は本来 partial 読みに固執していない。Codex CLI の prompt が「partial で十分」と言っているから、そこで止まっているだけだ。

## 仕様の側はどう言っているか

[Agent Skills 仕様](https://agentskills.io/specification#progressive-disclosure) は activation 時に「the entire file」をロードすると書いており、SKILL.md 本体の推奨上限は **500 行 / 5000 tokens**。Codex CLI の prompt が言う「partial で十分」とは正面から矛盾している。

前回の記事で書いた「だいたい 200 行に収まる」「だから 220 は仕様どおり」という説明は、仕様の許容ライン (500 行) を半分以下に見積もった誤読だった。

## Quaere 側で言えること

前回の記事で「SKILL.md 本体は 200 行以内に収める」という助言を書いた。これは今でも実用上は使えるが、扱いは変わる:

- 仕様 (500 行) に合わせる助言ではなく、Codex CLI の prompt 由来の現場 cap に合わせる workaround。具体的な cap は gpt-5.5 で 220、gpt-5.4 で 260。
- 上流の wording が直れば、この助言は要らなくなる。
- Codex 以外の harness (Claude Code、OpenCode、claudex 経路) では最初から要らない。

Quaere の skill 群を 200 行以内に書き直す仕事は宿題として残っている。書き直してから再 eval すると、現在 README にある eval 数字 (+37.7pp / +8.7pp) が「Codex が後半を読まないまま走った結果」なのか「最初の 200 行で実質足りていた」のかが、ようやく分けて測れる。

## 計測のソース

raw data と再現手順は gist に置いてある。cap-test skill と Codex セッションログの直接 grep で確認できる。

- 計測ノート (JP): https://gist.github.com/haru0416-dev/8c1b01098f46e29d244f2085e408c789
- English version: https://gist.github.com/haru0416-dev/4fd584e8616698cc1cc34c04b03f70ee

[前回の記事](https://zenn.dev/haru0416/articles/codex-skill-md-220-lines)では計測の生データだけを出し、「なぜ 220 なのか」の解釈は推測込みで、結果として何箇所か間違えた。今回見つかった render.rs の 1 行 + Issue #16479 が、その推測を書き直してくれる。
