# feat : Lv 1. Docker로 MySQL과 Redis 설정
- Docker로 MySQL과 Redis를 실행
- Spring 애플리케이션의 환경 변수를 설정

# feat : Lv 2. SQL을 JPA 인덱스로 표현하기
- SQL(CREATE INDEX idx_chat_world_created_at ON chat_messages(world_id, created_at);)과 같은 인덱스가 생성되도록 ChatMessage의 @Table을 수정
  (SQL을 직접 실행하는 대신 @Table과 @Index로 인덱스를 선언)

# feat : Lv 3. 요청 검증과 DTO: 플레이어 등록
- API 명세에 맞게
  플레이어 등록 Controller(Controller의 요청 매핑, JSON 본문 바인딩, DTO 검증과 성공 응답),
  요청 DTO(닉네임은 비어 있지 않은 2~12글자이며, 영문 대소문자와 숫자, 밑줄만 허용)와
  서비스(이미 등록된 닉네임이면 ConflictException으로 DUPLICATE_NICKNAME 에러, 중복이 아니면 제공된 savePlayer(new Player(request.getNickname()))로 저장)를 구현
- 테스트 확인(주석 해제)

# feat : Lv 4. 월드 생성 
- worldOperations.duringCreation()에 람다를 전달하고,
  람다 안에서 worldRepository.countRootWorlds()가 MAX_WORLDS 이상이면 throw new ConflictException("WORLD_LIMIT_REACHED") 에러를 던지고,
  제한을 넘지 않으면 createPreparedWorld(request)의 결과를 반환
- 테스트 확인(주석 해제)

# feat : Lv 5. 채팅 저장과 내역 조회
- 채팅 한 건을 DB에 저장하는 기능 구현
  (worldId로 월드를 조회, 없으면 NotFoundException으로 WORLD_NOT_FOUND로 예외처리,
  전달받은 월드 ID, 보낸 사람의 닉네임, 채팅 내용를 이용해 ChatMessage 생성, chatMessageRepository.save()로 저장
  저장 결과를 saved에 담아 savedResponse(worldId, saved)의 결과를 반환)
- 최근 채팅을 대화 순서대로 반환하는 기능 구현
  (recent를 reversed()를 통해 오래된 순서로 바꾸고 ChatMessageResponse DTO 목록으로 반환)
- 테스트 확인(주석 해제)

# feat : Lv 6. 최근 채팅 조회 API 구현
- chats()에 GET /worlds/{worldId}/chats 요청을 매핑
- URL의 worldId와 limit을 매개변수로 받고,
  limit을 생략했을 때의 기본값(@RequestParam(defaultValue = "50")) 설정
- chatService.getRecentMessages(worldId, limit)을 호출하여 최근 채팅 목록을 조회,
  결과를 명세의 성공 상태 코드와 함께 반환
- 테스트 확인(주석 해제)

# feat : Lv 7. WebSocket 연결과 사용자 식별
- playerRepository.findByNickname(nickname)으로 플레이어를 조회해 player에 대입, 조회 결과가 없으면 null 사용
- worldRepository.findById(worldId)로 월드를 조회해 world에 대입, 조회 결과가 없으면 null 사용
- attributes에 ATTR_NICKNAME을 키로 nickname, ATTR_WORLD_ID를 키로 worldId 저장
- 테스트 확인(주석 해제)

# feat : Lv 8. HandshakeInterceptor 등록
- /ws/worlds/{worldId} 경로의 핸들러에 NicknameHandshakeInterceptor를 등록
- 테스트 확인(주석 해제)

# feat : Lv 9. 월드별 WebSocket 세션 관리
- register()에서 sessions.putIfAbsent(nicknameKey, candidate)로 연결을 등록,
  반환값이 null이면 새로 등록한 것이므로 added를 true로 설정
- get()에서 sessions.get(key(nickname))으로 연결을 조회해 반환
- 테스트 확인(주석 해제)

# feat : Lv 10. Redis 접속 상태 관리
- join()에서 redisTemplate.opsForZSet().add(key, connectionId, expiresAt())로 접속 정보를 저장
- leave()에서 redisTemplate.opsForZSet().remove(key(worldId), connectionId)로 종료된 연결을 삭제
- 테스트 확인(주석 해제)

# feat : Lv 11. 메시지 라우팅과 Ping/Pong
- MessageRouter.route()에서 찾아 둔 handler의 handle(context, message) 호출
- PingWsHandler에서 presenceService.heartbeat(context.worldId(), connection.connectionId()) 호출해 현재 연결의 Redis 접속 상태를 갱신
- broadcaster.sendTo(context.session(), new PongResponse())로 ping을 보낸 연결에 pong을 응답
- 테스트 확인(주석 해제)

# feat : Lv 12. 플레이어 이동 요청 처리
- 클라이언트가 보낸 이동 메시지를 PlayerAction.Move 객체로 만들어 게임 엔진에 전달
  (PlayerAction.Move의 생성자 순서는 nickname, x, y, z, yaw, pitch, crouching, gliding, finalSceneActionId, 마지막 인자는 메서드에서 미리 구해 놓은 finalSceneActionId를 그대로 전달)
- 읽은 값으로 PlayerAction.Move를 생성하고 engineManager.enqueue(월드 ID, 이동 요청)에 전달, 현재 월드 ID는 context.worldId(), 닉네임은 context.nickname()을 사용
- 테스트 확인(주석 해제)

# feat : Lv 13. 채팅 요청 처리와 응답 구성
- readContent()에서 WsFields.text(메시지, 필드명)를 사용하여 명세의 content 필드를 읽도록 구현
- ChatResponse의 sender, content, timestamp 필드와 생성자를 구현하고, type 필드값 "chat"으로 설정
- createResponse()에서 chatService.saveMessage()를 호출,
  월드 ID(context.worldId()), 닉네임(context.nickname()), 채팅 내용(content)을 전달,
  저장 결과를 ChatResponse로 변환하여 반환
- 테스트 확인(주석 해제)

# feat : Lv 14. 같은 월드의 참여자에게 채팅 전송
- WorldBroadcaster.broadcast(worldId, message)로 같은 월드의 세션에 메시지를 전달(보낸 사람도 수신 대상에 포함)
- 테스트 확인(주석 해제)