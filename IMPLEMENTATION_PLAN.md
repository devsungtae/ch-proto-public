# Proto 구현 계획: UserChat, Group, ChatMessage, ChatThread, ChatSession, MeetMessage

## 개요

v5 open API의 나머지 리소스 6개에 대한 proto 정의를 **단일 오퍼레이션, 단일 관심사** 원칙으로 작성한다.

### 핵심 원칙

- **단일 오퍼레이션**: 하나의 proto API는 하나의 작업만 수행한다
- **단일 관심사 모델**: 응답에는 해당 오퍼레이션의 주 모델만 포함한다 (관련 모델을 함께 반환하지 않는다)
- **v6 open API 조합**: v6 open API 레이어에서 여러 proto API를 조합하여 복합 응답을 구성한다

### 기존 v5 복합 응답 → v6 단일 관심사 분리 예시

```
v5: GET /user-chats/{id}
  → LegacyV5UserChatView: userChat + user + message + campaign + chatTags + session + ...

v6: 여러 proto API 조합
  → GetUserChat       → UserChat만
  → GetUser           → User만 (기존 proto 재사용)
  → SearchChatTags    → ChatTag만 (기존 proto 재사용)
  → ...

v5: GET /groups/{id}
  → GroupView: group + managers + bookmark + session

v6: 여러 proto API 조합
  → GetGroup           → Group만
  → SearchChatSessions → ChatSession만 (공유 proto 재사용)
  → ...
```

---

## 대상 리소스

| v5 Resource | 경로 | 비고 |
|---|---|---|
| **UserChatsResource** | `/open/v5/user-chats` | 주 리소스 |
| **UserChatsForUserResource** | `/open/v5/users/{userId}/user-chats` | 주 리소스 |
| **GroupsResource** | `/open/v5/groups` | 주 리소스 |
| **ChatMessagesResource** | `.../{chatId}/messages` | 공유 sub-resource (UserChats, Groups 양쪽에서 사용) |
| **ChatThreadsResource** | `.../groups/{groupId}/threads/{messageId}` | Groups 전용 sub-resource |
| **MeetMessagesResource** | `.../user-chats/{userChatId}/meets` | UserChats 전용 sub-resource |

### 제외 대상

| 엔드포인트 | 사유 |
|---|---|
| `DELETE /user-chats/{id}/remove` | deprecated (2022.11.01), v6에서 제거 |
| `GET /user-chats/cases` | deprecated (2026.02.25), 2026-04-30 제거 예정 |

---

## Phase 1: Model Proto

### 1-1. `coreapi/model/message.proto` (신규)

close된 `user-chat-proto` 브랜치의 내용을 기반으로 작성한다.

**정의할 타입:**

| 타입 | 설명 |
|---|---|
| `MessageState` enum | SENDING, SENT, FAILED, REMOVED |
| `MessageWritingType` enum | STANDARD, CUSTOM, EMAIL |
| `LogAction` enum | CHANGE_NAME, CLOSE, OPEN, ASSIGN, ... (42개 값) |
| `MessageLog` message | 시스템 로그 메시지 데이터 |
| `MessageReaction` message | 이모지 반응 데이터 |
| `MessageThread` message | 스레드 메타데이터 (thread가 있는 root message에 포함됨) |
| `Message` message | 메시지 엔티티 (23개 필드) |

**참고**: `message_content.proto`는 이미 main에 존재 (Block, MessageButton, MessageFile, MessageWebPage, MessageOption).

**close된 브랜치 대비 변경 사항**: 없음. 모델 자체는 이미 잘 작성되어 있다.

### 1-2. `coreapi/model/user_chat.proto` (신규)

close된 `user-chat-proto` 브랜치의 내용을 기반으로 작성한다.

**정의할 타입:**

| 타입 | 설명 |
|---|---|
| `UserChatState` enum | OPENED, CLOSED, SNOOZED, QUEUED, INITIAL, MISSED |
| `UserChat` message | UserChat 엔티티 (39개 필드) |

**close된 브랜치 대비 변경 사항**: 없음. 모델 자체는 이미 잘 작성되어 있다.

### 1-3. `coreapi/model/chat_session.proto` (신규 작성)

ChatSession 모델을 새로 정의한다.

**참조할 Java 모델**: `io.channel.api.models.chat.session.ChatSession`

**정의할 타입:**

| 타입 | 설명 |
|---|---|
| `SessionWatch` enum | 세션 알림 감시 설정 |
| `ChatSession` message | 채팅 세션 (채팅 참여 멤버를 나타냄) |

**노출할 필드 (20개):**

v5 공개 API 스펙의 모든 필드를 지원한다.

| # | Proto 필드 | 타입 | v5 JSON 필드 | 설명 |
|---|---|---|---|---|
| 1 | `id` | string | `id` | 고유 식별자 (computed: `key-chatId`) |
| 2 | `key` | string | `key` | 세션 키 (DynamoDB hash key) |
| 3 | `channel_id` | string | `channelId` | 채널 ID |
| 4 | `chat_type` | string | `chatType` | 채팅 유형 (userChat, group 등) |
| 5 | `chat_id` | string | `chatId` | 참여 중인 채팅 ID |
| 6 | `chat_key` | string | `chatKey` | 채팅 키 |
| 7 | `person_type` | string | `personType` | 참여자 유형 (manager, user 등) |
| 8 | `person_id` | string | `personId` | 참여자 ID |
| 9 | `updated_key` | string | `updatedKey` | 업데이트 정렬 키 |
| 10 | `unread_key` | string | `unreadKey` | 미읽음 정렬 키 |
| 11 | `alert` | int32 | `alert` | 알림 카운트 |
| 12 | `unread` | int32 | `unread` | 미읽음 카운트 |
| 13 | `watch` | SessionWatch | `watch` | 알림 감시 설정 |
| 14 | `all_mention_important` | bool | `allMentionImportant` | @all 멘션 중요 여부 |
| 15 | `read_at` | Timestamp | `readAt` | 마지막 읽은 시각 |
| 16 | `received_at` | Timestamp | `receivedAt` | 마지막 수신 시각 |
| 17 | `posted_at` | Timestamp | `postedAt` | 마지막 발신 시각 |
| 18 | `updated_at` | Timestamp | `updatedAt` | 업데이트 시각 |
| 19 | `created_at` | Timestamp | `createdAt` | 생성 시각 |
| 20 | `version` | int64 | `version` | 엔티티 버전 |

**제외할 필드 (v5 스펙에 없음):**

| Java 필드 | 제외 사유 |
|---|---|
| `teamChatSectionId` | v5 공개 API 스펙에 포함되지 않음. 팀챗 섹션 내부 라우팅용 |

### 1-4. `coreapi/model/group.proto` (신규 작성)

Group 모델을 새로 정의한다.

**참조할 Java 모델**: `io.channel.api.models.Group`

**정의할 타입:**

| 타입 | 설명 |
|---|---|
| `GroupScope` enum | ALL, PUBLIC, PRIVATE |
| `Group` message | 그룹(팀챗) 엔티티 |

**노출할 필드:**

| # | Proto 필드 | 타입 | Java 원본 | 설명 |
|---|---|---|---|---|
| 1 | `id` | string | `id` | 고유 그룹 식별자 |
| 2 | `channel_id` | string | `channelId` | 채널 ID |
| 3 | `title` | string | `title` | 그룹 이름 (채널 내 유일) |
| 4 | `scope` | GroupScope | `scope` | 그룹 공개 범위 |
| 5 | `manager_ids` | repeated string | `managerIds` | 그룹 멤버 매니저 ID 목록 |
| 6 | `icon` | string | `icon` | 그룹 아이콘 |
| 7 | `live_meet_id` | string | `liveMeetId` | 진행 중인 미팅 ID |
| 8 | `description` | string | `description` | 그룹 설명 |
| 9 | `created_at` | Timestamp | `createdAt` | 생성 시각 |
| 10 | `updated_at` | Timestamp | `updatedAt` | 업데이트 시각 |

---

## Phase 2: Service Proto

### 2-1. `coreapi/service/user_chat.proto` (신규)

UserChat 모델에 대한 CRUD + 상태 관리 오퍼레이션만 포함한다.
메시지, 세션 등 다른 관심사의 오퍼레이션은 포함하지 않는다.

**v5 UserChatsResource + UserChatsForUserResource → proto 매핑:**

| # | v5 엔드포인트 | Proto 메시지 | Result 내용 |
|---|---|---|---|
| 1 | `GET /user-chats` (managed) | `SearchUserChats{Request,Result}` | `repeated UserChat` + pagination |
| 2 | `GET /user-chats/{id}` | `GetUserChat{Request,Result}` | `UserChat` |
| 3 | `PUT /user-chats/{id}/open` | `OpenUserChat{Request,Result}` | `UserChat` |
| 4 | `PUT /user-chats/{id}/snooze` | `SnoozeUserChat{Request,Result}` | `UserChat` |
| 5 | `PATCH /user-chats/{id}/close` | `CloseUserChat{Request,Result}` | `UserChat` |
| 6 | `PATCH /user-chats/{id}/invite` | `InviteManagersToUserChat{Request,Result}` | `UserChat` |
| 7 | `PATCH /user-chats/{id}/assign-to/managers/{managerId}` | `AssignManagerToUserChat{Request,Result}` | `UserChat` |
| 8 | `PATCH /user-chats/{id}` (update) | `UpdateUserChat{Request,Result}` | `UserChat` |
| 9 | `DELETE /user-chats/{id}` | `DeleteUserChat{Request,Result}` | 빈 메시지 |
| 10 | `GET /users/{userId}/user-chats` | `SearchUserChatsForUser{Request,Result}` | `repeated UserChat` + pagination |
| 11 | `POST /users/{userId}/user-chats` | `CreateUserChat{Request,Result}` | `UserChat` |

**close된 브랜치 대비 변경 사항:**
- `SearchUserChatMessages`, `CreateUserChatMessage` **제거** → `chat_message.proto`로 이동 (관심사 분리)
- 나머지는 이미 단일 관심사로 되어 있어 그대로 사용 가능

### 2-2. `coreapi/service/group.proto` (신규 작성)

Group 모델에 대한 조회/수정 오퍼레이션.
메시지, 스레드, 세션 등 다른 관심사의 오퍼레이션은 포함하지 않는다.

**v5 GroupsResource → proto 매핑:**

| # | v5 엔드포인트 | Proto 메시지 | Result 내용 |
|---|---|---|---|
| 1 | `GET /groups` | `SearchGroups{Request,Result}` | `repeated Group` + pagination |
| 2 | `GET /groups/{groupId}` | `GetGroup{Request,Result}` | `Group` |
| 3 | `GET /groups/@{groupName}` | `GetGroupByName{Request,Result}` | `Group` |
| 4 | `PATCH /groups/{groupId}` | `UpdateGroup{Request,Result}` | `Group` |
| 5 | `PATCH /groups/@{groupName}` | `UpdateGroupByName{Request,Result}` | `Group` |
| 6 | `GET /groups/{groupId}/sessions` | `SearchChatSessions` (chat_session.proto 재사용) | `repeated ChatSession` |

**설계 근거:**
- v5에서 ID와 name으로 각각 조회/수정할 수 있으므로 별도 오퍼레이션으로 정의
- GroupView의 복합 데이터(managers, bookmark, session)는 단일 관심사 원칙에 따라 제외. Group만 반환
- `GET /groups/{id}/sessions`는 `chat_session.proto`의 `SearchChatSessions`를 재사용 (chat_type=group)
- sub-resource 위임(messages, threads)은 chat_message.proto, chat_thread.proto에서 커버

**Request 필드 참고:**
- SearchGroups: `channel_id`, `cursor`, `limit`
- GetGroup: `channel_id`, `group_id`
- GetGroupByName: `channel_id`, `group_name`
- UpdateGroup: `channel_id`, `group_id`, `bot_name`, 수정 가능 필드 (title, scope, icon, description)
- UpdateGroupByName: `channel_id`, `group_name`, `bot_name`, 수정 가능 필드

### 2-3. `coreapi/service/chat_message.proto` (신규 작성)

채팅 메시지 오퍼레이션. `chat_type` + `chat_id`를 파라미터로 받아 UserChat과 Group 양쪽에서 사용 가능하게 한다.

**v5 ChatMessagesResource → proto 매핑:**

| # | v5 엔드포인트 | Proto 메시지 | Result 내용 |
|---|---|---|---|
| 1 | `GET .../messages` | `SearchChatMessages{Request,Result}` | `repeated Message` + pagination |
| 2 | `POST .../messages` | `CreateChatMessage{Request,Result}` | `Message` |

**Request 공통 필드:**
```
channel_id, chat_type, chat_id
```

**설계 근거:**
- ChatMessagesResource는 Chat 인터페이스를 받는 공유 sub-resource (UserChats와 Groups 모두 사용)
- proto도 generic하게 `chat_type` + `chat_id`를 사용하여 채팅 유형에 무관하게 동작
- close된 브랜치의 `SearchUserChatMessages`/`CreateUserChatMessage`를 `user_chat.proto`에서 분리하여 이 파일로 이동

### 2-4. `coreapi/service/media.proto` (신규 작성)

파일/미디어 관련 오퍼레이션.

**v5 엔드포인트 → proto 매핑:**

| # | v5 엔드포인트 | Proto 메시지 | Result 내용 |
|---|---|---|---|
| 1 | `GET .../messages/file` (ChatMessagesResource) | `GetSignedFileUrl{Request,Result}` | `string url` |

**설계 근거:**
- 파일 서명 URL 발급은 채팅 메시지 CRUD와는 다른 관심사이므로 별도 서비스로 분리
- GetSignedFileUrl: `channel_id`, `chat_type`, `chat_id`, `key`
- `GetMeetRecording`은 이미 `meet.proto`에 정의되어 있으므로 여기에 포함하지 않음

### 2-5. `coreapi/service/chat_thread.proto` (신규)

스레드 오퍼레이션. Groups 전용이지만 proto는 `chat_type` + `chat_id`로 범용 설계한다.

**v5 ChatThreadsResource → proto 매핑:**

| # | v5 엔드포인트 | Proto 메시지 | Result 내용 |
|---|---|---|---|
| 1 | `GET .../threads/{messageId}` (show) | `GetChatThread{Request,Result}` | `Message` (thread 메타데이터 포함) |
| 2 | `POST .../threads/{messageId}` (create) | `CreateChatThread{Request,Result}` | `Message` |
| 3 | `GET .../threads/{messageId}/messages` | `SearchChatThreadMessages{Request,Result}` | `repeated Message` + pagination |
| 4 | `POST .../threads/{messageId}/messages` | `CreateChatThreadMessage{Request,Result}` | `Message` |

**close된 브랜치 대비 변경 사항:**
- `GetChatThreadResult`에서 `bot`, `managers` 필드 **제거** → Message만 반환
- `CreateChatThreadResult`에서 `bot`, `managers` 필드 **제거** → Message만 반환
- Message 모델에 이미 `MessageThread thread` 필드가 포함되어 있으므로, thread 메타데이터는 Message를 통해 접근
- v6 open API에서 bot 정보가 필요하면 `Message.person_id` → `GetBot` proto를 별도 호출
- v6 open API에서 manager 목록이 필요하면 `MessageThread.manager_ids` → 별도 manager 조회

### 2-6. `coreapi/service/chat_session.proto` (신규 작성)

채팅 세션 조회 오퍼레이션.

**v5 엔드포인트 → proto 매핑:**

| # | v5 엔드포인트 | Proto 메시지 | Result 내용 |
|---|---|---|---|
| 1 | `GET /user-chats/{id}/sessions` (UserChatsResource) | `SearchChatSessions{Request,Result}` | `repeated ChatSession` |
| 2 | `GET /groups/{id}/sessions` (GroupsResource) | (동일 proto 재사용) | |

**Request 필드:**
```
channel_id, chat_type, chat_id
```

**참고:** v5에서 pagination이 없음 (`sessionDao.findAll(chat)` 호출). proto에서도 pagination 없이 전체 목록 반환.

### ~~2-7. `coreapi/service/meet_message.proto`~~ (기존 proto로 대체)

`SearchMeetMessages`와 `GetMeetRecording`은 이미 `coreapi/service/meet.proto`에 정의되어 있다.
별도 파일을 생성하지 않는다.

---

## Phase 3: 검증 및 코드 생성

1. `make lint` — buf lint 통과 확인
2. `make generate` — Go/Java 코드 생성
3. 생성된 코드 커밋

---

## 파일 생성 요약

| Phase | 파일 경로 | 유형 | 기반 |
|---|---|---|---|
| 1-1 | `coreapi/model/message.proto` | 신규 | close된 브랜치 재사용 |
| 1-2 | `coreapi/model/user_chat.proto` | 신규 | close된 브랜치 재사용 |
| 1-3 | `coreapi/model/chat_session.proto` | 신규 | 새로 작성 |
| 1-4 | `coreapi/model/group.proto` | 신규 | 새로 작성 |
| 2-1 | `coreapi/service/user_chat.proto` | 신규 | close된 브랜치 기반 수정 (메시지 오퍼레이션 분리) |
| 2-2 | `coreapi/service/group.proto` | 신규 | 새로 작성 |
| 2-3 | `coreapi/service/chat_message.proto` | 신규 | 새로 작성 (generic chat_type + chat_id) |
| 2-4 | `coreapi/service/media.proto` | 신규 | 새로 작성 (GetSignedFileUrl + GetMeetRecording) |
| 2-5 | `coreapi/service/chat_thread.proto` | 신규 | close된 브랜치 기반 수정 (Result에서 bot/managers 제거) |
| 2-6 | `coreapi/service/chat_session.proto` | 신규 | 새로 작성 |
| ~~2-7~~ | ~~`coreapi/service/meet_message.proto`~~ | ~~신규~~ | 기존 `meet.proto`에 이미 존재 |

---

## 기존 proto 재활용

다음 proto는 이미 main에 존재하며, v6 open API 조합 시 재활용한다:

| 기존 Proto | 용도 |
|---|---|
| `service/bot.proto` | GetBot — Message의 bot 정보 조회 시 |
| `service/manager.proto` | GetManager — Thread의 manager 목록 조회 시 |
| `service/user.proto` | GetUser — UserChat의 user 정보 조회 시 |
| `service/chat_tag.proto` | SearchChatTags — UserChat의 태그 조회 시 |
| `model/message_content.proto` | Message 모델이 참조 (Block, MessageButton 등) |

---

## v5 → proto 전체 매핑 (엔드포인트 26개)

### UserChatsResource (10개 오퍼레이션 + 2 deprecated + 2 위임)

| v5 | HTTP | Proto | 파일 |
|---|---|---|---|
| `/user-chats` | GET | SearchUserChats | user_chat.proto |
| `/user-chats/{id}` | GET | GetUserChat | user_chat.proto |
| `/user-chats/{id}/open` | PUT | OpenUserChat | user_chat.proto |
| `/user-chats/{id}/snooze` | PUT | SnoozeUserChat | user_chat.proto |
| `/user-chats/{id}/close` | PATCH | CloseUserChat | user_chat.proto |
| `/user-chats/{id}/invite` | PATCH | InviteManagersToUserChat | user_chat.proto |
| `/user-chats/{id}/assign-to/managers/{mid}` | PATCH | AssignManagerToUserChat | user_chat.proto |
| `/user-chats/{id}` | PATCH | UpdateUserChat | user_chat.proto |
| `/user-chats/{id}` | DELETE | DeleteUserChat | user_chat.proto |
| `/user-chats/{id}/sessions` | GET | SearchChatSessions | chat_session.proto |
| `/user-chats/{id}/remove` | DELETE | ~~제거~~ | deprecated |
| `/user-chats/cases` | GET | ~~제거~~ | deprecated |

### UserChatsForUserResource (2개 오퍼레이션)

| v5 | HTTP | Proto | 파일 |
|---|---|---|---|
| `/users/{uid}/user-chats` | GET | SearchUserChatsForUser | user_chat.proto |
| `/users/{uid}/user-chats` | POST | CreateUserChat | user_chat.proto |

### GroupsResource (5개 오퍼레이션 + 1 공유 + 4 위임)

| v5 | HTTP | Proto | 파일 |
|---|---|---|---|
| `/groups` | GET | SearchGroups | group.proto |
| `/groups/{groupId}` | GET | GetGroup | group.proto |
| `/groups/@{groupName}` | GET | GetGroupByName | group.proto |
| `/groups/{groupId}` | PATCH | UpdateGroup | group.proto |
| `/groups/@{groupName}` | PATCH | UpdateGroupByName | group.proto |
| `/groups/{groupId}/sessions` | GET | SearchChatSessions | chat_session.proto (재사용) |

### ChatMessagesResource (2개 오퍼레이션 + 1 미디어)

| v5 | HTTP | Proto | 파일 |
|---|---|---|---|
| `.../{chatId}/messages` | GET | SearchChatMessages | chat_message.proto |
| `.../{chatId}/messages` | POST | CreateChatMessage | chat_message.proto |
| `.../{chatId}/messages/file` | GET | GetSignedFileUrl | media.proto |

### ChatThreadsResource (4개 오퍼레이션)

| v5 | HTTP | Proto | 파일 |
|---|---|---|---|
| `.../threads/{msgId}` | GET | GetChatThread | chat_thread.proto |
| `.../threads/{msgId}` | POST | CreateChatThread | chat_thread.proto |
| `.../threads/{msgId}/messages` | GET | SearchChatThreadMessages | chat_thread.proto |
| `.../threads/{msgId}/messages` | POST | CreateChatThreadMessage | chat_thread.proto |

### MeetMessagesResource (기존 meet.proto에 이미 존재)

| v5 | HTTP | Proto | 파일 |
|---|---|---|---|
| `.../meets/{msgId}/messages` | GET | SearchMeetMessages | meet.proto (기존) |
| `.../meets/{msgId}/recording` | GET | GetMeetRecording | meet.proto (기존) |

---

## PR 분할 전략

독립적인 4개 PR로 나눈다. 각 PR은 model + service를 함께 포함하여 단독으로 리뷰/머지할 수 있다.
PR 간 의존성이 없으므로 병렬로 작업 가능하다. 단, PR1(Message)이 다른 PR에서 Message 모델을 import하므로 먼저 머지하는 것을 권장한다.

### PR1: Message 도메인 (가장 먼저 머지 권장)

**파일:**
- `coreapi/model/message.proto` — Message, MessageThread, MessageLog, MessageReaction, enums
- `coreapi/service/chat_message.proto` — SearchChatMessages, CreateChatMessage (2 ops)
- `coreapi/service/chat_thread.proto` — GetChatThread, CreateChatThread, SearchChatThreadMessages, CreateChatThreadMessage (4 ops)

**오퍼레이션 수:** 6개
**근거:** chat_message, chat_thread 모두 Message 모델에 의존. Message 모델과 함께 묶는 것이 자연스럽다.
**의존성:** message_content.proto (기존 main)
**참고:** SearchMeetMessages, GetMeetRecording은 이미 `meet.proto`에 정의되어 있으므로 별도 작업 불필요.

### PR2: UserChat 도메인

**파일:**
- `coreapi/model/user_chat.proto` — UserChat, UserChatState
- `coreapi/service/user_chat.proto` — 11개 UserChat 오퍼레이션

**오퍼레이션 수:** 11개
**근거:** UserChat model + service가 자체 완결적. close된 브랜치에서 메시지 오퍼레이션만 분리하면 됨.
**의존성:** 없음 (독립적)

### PR3: Group 도메인

**파일:**
- `coreapi/model/group.proto` — Group, GroupScope
- `coreapi/service/group.proto` — SearchGroups, GetGroup, GetGroupByName, UpdateGroup, UpdateGroupByName (5 ops)

**오퍼레이션 수:** 5개
**근거:** Group model + service가 자체 완결적.
**의존성:** 없음 (독립적)

### PR4: ChatSession + Media

**파일:**
- `coreapi/model/chat_session.proto` — ChatSession, SessionWatch
- `coreapi/service/chat_session.proto` — SearchChatSessions (1 op)
- `coreapi/service/media.proto` — GetSignedFileUrl (1 op)

**오퍼레이션 수:** 2개
**근거:** ChatSession은 소규모 모델+서비스. Media도 소규모. 별도 PR로 만들기엔 너무 작으므로 합친다.
**의존성:** 없음 (독립적)
**참고:** GetMeetRecording은 이미 `meet.proto`에 정의되어 있으므로 media.proto에는 GetSignedFileUrl만 포함.

### PR 머지 순서

```
PR1 (Message) ──────→ 먼저 머지 (다른 도메인이 Message import 가능성)
PR2 (UserChat) ─────→ PR1 이후 또는 병렬
PR3 (Group) ────────→ PR1 이후 또는 병렬
PR4 (Session+Media) → 순서 무관
```

### 참고: SearchGroups 정렬

v5에서 Groups 목록은 name 기준 오름차순 고정 (sortOrder 파라미터 없음). proto Request에 sort 필드를 추가하지 않고 서버 측에서 name 오름차순을 기본 동작으로 한다.
