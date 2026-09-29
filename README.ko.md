# herdr-pane-mru

[English](README.md) | **한국어**

[herdr](https://herdr.dev) 의 방향 이동(`prefix+h/j/k/l`)이 tmux 처럼 **최근에 있던 pane** 으로 돌아가게 하는 플러그인.

## 왜 필요한가

herdr 0.9.0 의 방향 이동(`src/layout.rs` `find_in_direction`)은 화면 기하만으로 대상을 고른다.

```
정렬 키 = (경계 거리, 겹침이 큰 쪽, 중심 거리, 레이아웃 순서)
```

아래처럼 한쪽 열이 둘로 나뉜 레이아웃에서 `B` 에 있다가 `C` 로 갔다 돌아오면, 앞의 세 기준이 모두 비겨
**레이아웃 순서가 앞선 `A` 로 간다.** 방금 있던 `B` 를 기억하지 않는다.

```
┌─────┬─────┐
│  A  │     │
├─────┤  C  │     B → C → (왼쪽) ⇒ herdr: A    이 플러그인: B
│  B  │     │
└─────┴─────┘
```

tmux 는 이 경우 가장 최근에 쓴 pane 을 고른다. 이 플러그인은 그 동작을 재현한다.

## 동작

- `pane.focused` 이벤트 훅(`bin/record`)이 탭별 최근 사용 목록(MRU)을 `$HERDR_PLUGIN_STATE_DIR/<탭>.mru` 에 쌓는다.
  키보드·마우스·알림 이동 등 경로와 무관하게 모든 포커스 변화를 기록한다.
- 방향 액션(`bin/focus <dir>`)은 herdr 와 **같은 후보 규칙**(그 방향에 있고 수직축이 겹치는 pane)으로 후보를 추린 뒤
  정렬 키에 MRU 순위를 끼운다.

  ```
  herdr          = (경계 거리,           겹침↓, 중심 거리, 순서)
  herdr-pane-mru = (경계 거리, MRU 순위, 겹침↓, 중심 거리, 순서)
  ```

  경계 거리를 맨 앞에 둬서 **바로 붙은 이웃** 안에서만 이력을 따른다. 이력에 없는 후보끼리는 herdr 와 같은 결과가 나온다.
- herdr CLI 의 `pane focus` 는 방향만 받으므로, id 를 지정한 포커스는 소켓 API `pane.focus` 로 보낸다.
- zoom 상태이거나 무엇이든 실패하면 **herdr 기본 방향 이동으로 넘긴다** — 플러그인 문제로 이동 자체가 막히지 않게.

## 요구 사항

- herdr ≥ 0.9.0 (Linux / macOS)
- `bash`, `jq`, `socat`, `flock`(util-linux) — herdr 서버의 `PATH` 에서 찾는다. `jq`·`socat` 이 없으면 기본 이동으로 동작한다.

## 설치

```sh
herdr plugin install unstable-code/herdr-pane-mru
```

업데이트도 같은 명령을 다시 실행하면 된다. 원본 저장소는 자체 호스팅 GitLab 이고, `herdr plugin install` 이
GitHub 에서만 받으므로 [GitHub](https://github.com/unstable-code/herdr-pane-mru) 로 미러링한다.

개발용으로는 로컬 클론을 `link` 한다 — 작업트리를 그대로 쓰므로 `git pull` 이 곧 업데이트다.

```sh
git clone https://github.com/unstable-code/herdr-pane-mru.git
herdr plugin link ./herdr-pane-mru
```

`link` 로 연결해 둔 것을 `install` 로 바꾸려면 먼저 `herdr plugin unlink unstable-code.herdr-pane-mru` 를 해야 한다 —
herdr 는 로컬 경로로 연결된 플러그인을 install 로 덮어쓰지 않는다(herdr 0.9.0 `src/cli/plugin.rs` 의 `ensure_replacement_allowed`).

`~/.config/herdr/config.toml` 에서 내장 방향 이동의 키를 **비우고** 같은 키를 플러그인 액션에 건다.

```toml
[keys]
focus_pane_left = ""
focus_pane_down = ""
focus_pane_up = ""
focus_pane_right = ""

[[keys.command]]
key = "prefix+h"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-left"
description = "focus pane left (MRU)"

[[keys.command]]
key = "prefix+j"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-down"
description = "focus pane down (MRU)"

[[keys.command]]
key = "prefix+k"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-up"
description = "focus pane up (MRU)"

[[keys.command]]
key = "prefix+l"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-right"
description = "focus pane right (MRU)"
```

⚠️ 내장 키를 비우지 않으면 안 된다. `focus_pane_*` 를 사용자 설정에 적어 둔 상태에서 같은 키를 `[[keys.command]]` 에 걸면
**사용자 키끼리 충돌로 처리돼 플러그인 쪽 바인딩이 꺼진다**(herdr 0.9.0 `src/config/keybinds.rs` — 내장 액션을 먼저 등록한다).
`focus_pane_*` 를 아예 적지 않은 경우엔 기본값이 조용히 밀려나므로 비울 필요가 없다.

`description` 은 herdr 도움말(`prefix+?`)의 custom 그룹에 표시된다. 비우면 네 키가 모두 `custom command` 로만 보인다
(herdr 0.9.0 `src/input/keybind_help.rs`). 내장 도움말 문구(`focus pane left`)에 맞춰 두면 원래 무슨 키였는지 바로 읽힌다.

적용: `herdr server reload-config` (또는 설정해 둔 reload 키).

## 검증

격리된 herdr 서버(별도 `HOME`)에서 위 그림의 레이아웃을 만들고 확인했다(herdr 0.9.0).

| 시나리오 | 내장 이동 | 플러그인 |
|---|---|---|
| B → C → 왼쪽 | A | **B** |
| A → C → 왼쪽 | A | A |
| 그 방향에 pane 없음 | 이동 없음 | 이동 없음 |

플러그인 로그 기준 모든 명령 종료 코드 0, 소요 시간은 이동 약 34ms · 기록 약 23ms.

## 한계

- 키를 누를 때마다 셸·`herdr pane layout`·소켓 요청이 한 번씩 돌아 내장 이동보다 수십 ms 느리다.
- 포커스 이벤트마다 `bin/record` 프로세스가 뜬다.
- MRU 는 탭 단위다. pane 이 다른 탭으로 옮겨지면 새 id 를 받으므로 이전 이력은 이어지지 않는다.

## 제3자 저작물

`bin/focus` 의 후보 선정·정렬 규칙은 herdr 의 `find_in_direction`(`src/layout.rs`)
을 참고한 것이다. herdr 는 Apache-2.0 이다. herdr 의 코드나 바이너리를 재배포하지는 않는다.

## License

[MIT](LICENSE)
