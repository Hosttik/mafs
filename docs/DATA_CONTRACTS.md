# Спецификация данных между модулями, версия 0.1

Дата: 26 октября 2026. Это нормативное проектное описание и иллюстрации; исполняемые схемы, API-клиенты и компилятор ещё не реализованы.

## 1. Общие правила

Обмен — UTF-8 JSON. Все поля, перечисленные как обязательные, присутствуют; отсутствие знания выражается `null` или явным `unknown`, а не выдуманным значением. В будущей реализации — JSON Schema с закрытыми объектами и отдельные semantic validators. Совместимые добавления увеличивают minor schema version, изменение смысла/единиц — major.

Все выходные артефакты используют оболочку:

| Поле | Тип | Требование |
|---|---|---|
| schema_version | string | Например `mafs/0.1` |
| artifact_type | enum/string | Один из описанных ниже типов |
| artifact_id | string | Стабильный UUID или локальный уникальный id |
| revision | integer ≥ 1 | Новая редакция не меняет предыдущую |
| production_id | string | Общий id производства |
| parent_refs | ArtifactRef[] | Точные версии входных артефактов; для корня пустой массив |
| created_at | string | ISO-8601 UTC |
| producer | object | module, code_revision; для AI также ModelInvocationRef |
| content_sha256 | string | SHA-256 канонического payload; вычисляет код |
| payload | object | Типизированное содержимое |

`ArtifactRef = {artifact_id, revision, content_sha256}`. Ссылки должны разрешаться в manifest; stale revision не допускается. Канонический JSON для хеша фиксируется единственным сериализатором: стабильные ключи, числа, UTF-8; timestamp оболочки в payload hash не включается.

`AssetRef = {asset_id, sha256, media_type, private_uri, rights_record_id}`. Внешние модели и референсные изображения не передаются между этапами в виде произвольного публичного URL. URI с секретами не пишется в журналы.

Идентификаторы `character_id → beat_id → scene_id → shot_id → take_id → final time range` образуют прослеживаемую цепочку. Новая попытка получает новый take_id, но сохраняет shot_id.

### Координаты и время

| Область | Единицы |
|---|---|
| Референс | `start_us`, `end_us`: целые микросекунды относительно source presentation timeline; `pts` и `time_base={num,den}` сохраняются у evidence |
| Сценарий | Оценка длительности в целых миллисекундах; это ещё не расписание движка |
| Постановка/исполнение | Целые production ticks, `tick_rate={num:20,den:1}` для первого backend |
| Рендер | `fps={num:30,den:1}`, целый frame_index от 0; вычисления через рациональные числа |
| Звук | Целый sample_index, sample_rate=48000; связь с production tick и final frame |
| Позиция | Minecraft blocks, x/y/z; y вверх; у актёра origin в ногах |
| Углы | Градусы; canonical yaw=0 в +Z, yaw=90 в −X, pitch положительный вниз; roll задаётся только камерой |

Все интервалы полуоткрытые `[start,end)`. `N` кадров покрывают `[0,N/fps)`; последний кадр в `(N−1)/fps`. RenderProfile требует длительность, кратную выходному кадру. Компилятор не округляет независимо каждую склейку: финальный таймлайн строится в целых кадрах.

У референсов интервалы могут иметь переменную частоту кадров; timestamp прокси обязательно отображается на исходный PTS. `frame_no / nominal_fps` для VFR не считается достоверным timecode.

Canonical yaw не равен внутреннему yaw BBS. Преобразование выполняется и проверяется только backend-адаптером. CameraSpec задаёт также `target`, поэтому независимая проверка может обнаружить ошибку угла.

## 2. ReferenceAsset и AnalysisPolicy

**ReferenceAsset:** source_id, source_kind=`local_mp4`, private AssetRef, origin_url|null, creator_label|null, reference_role=`storytelling|cinematography`, rights_record, duration_us, streams[], original_time_base, proxy_map_ref|null.

**AnalysisPolicy:** выбранные интервалы, sampling base/max, размер кадра/detail, размер/overlap окна, лимит кадров/audio duration, model_policy_refs, разрешённый budget_grant_id, held_out flag. Не переданный grant означает только локальный dry-run без платных запросов.

`rights_record` фиксирует происхождение, кто предоставил файл и разрешённый workflow. Не означает автоматически права на публикацию или обучение.

## 3. ReferenceAnalysis и Evidence

**ReferenceAnalysis:** source_ref, policy_ref, shot_observations[], action_observations[], audio_events[], causal_hypotheses[], technique_candidates[], blind_intervals[], warnings[].

`Evidence = {source_id, source_sha256, start_us, end_us, frames: FrameEvidence[], audio: AudioEvidence[], coverage, gaps_us[]}`.

`FrameEvidence`: frame_id, pts, time_base, source_time_us, image AssetRef, crop|null. `AudioEvidence`: clip AssetRef, source_start_us, source_end_us, channel. `coverage` принимает `sampled|dense|continuous_signal`; sampled не означает непрерывный просмотр.

| Вывод/наблюдение | Обязательные поля |
|---|---|
| ShotObservation | shot_observation_id, evidence, boundary_kind, boundary_confidence, shot_size|null, camera_motion|null, visible_subjects[], observed_description |
| ActionObservation | observation_id, evidence, actor_local_ids[], observed_event, interpretation|null, alternative_interpretations[], observation_confidence, interpretation_confidence|null, technique_type, reproducibility |
| AudioEvent | audio_event_id, evidence, observed_signal, proposed_class|null, onset_us, timing_tolerance_us, confidence, related_observation_ids[] |
| CausalHypothesis | hypothesis_id, cause_observation_ids[], effect_observation_ids[], interpretation, supporting_evidence[], counter_evidence[], confidence, review_status |

`boundary_kind=hard_cut|fade|possible_cut|none|unknown`.

`reproducibility={status:supported|requires_tooling|unsupported|unknown, capability_ids[], constraints[], evidence_take_refs[]}`. Это инженерная оценка возможности постановки; отсутствие собственного теста не позволяет записать `render_verified`.

Уверенность — число 0–1, заданное моделью/разметчиком, с producer. Она не считается вероятностью, подтверждённой статистически. Скрытое намерение всегда находится в interpretation, даже при уверенности 0.99.

### Иллюстрация наблюдения

Ниже **синтетический пример**, не разметка реального ролика. Id намеренно не ссылается на существующий источник:

```json
{
  "observation_id": "illustration-only-obs-01",
  "source_id": "illustration-not-a-real-video",
  "start_us": 12000000,
  "end_us": 13800000,
  "observed_event": "A прекращает движение и поворачивает голову к B",
  "interpretation": "A ожидает реакции B",
  "alternative_interpretations": ["A следит за объектом позади B"],
  "observation_confidence": 0.92,
  "interpretation_confidence": 0.61,
  "technique_type": "reaction_pause",
  "reproducibility": {
    "status": "requires_tooling",
    "capability_ids": ["look_at", "hold_pose"],
    "constraints": ["голова должна быть различима в выбранном плане"],
    "evidence_take_refs": []
  }
}
```

Пример показывает payload-фрагмент; полный артефакт требует оболочку и Evidence. Валидатор не примет его как настоящее свидетельство без source/hash/frame refs.

## 4. TechniqueCard и TechniqueBundle

**TechniqueCard:** technique_id, kind=`story|staging|camera|editing|sound`, name, abstract_rule, intended_audience_effect, preconditions[], steps[], failure_modes[], counterexamples[], source_observation_refs[], abstraction_review, minecraft_mapping, evidence_status, validation_history[].

`minecraft_mapping={required_capabilities[], allowed_backends[], constraints[], implementation_status, own_test_take_refs[]}`.

`implementation_status=idea|implemented|render_verified|human_verified`; `evidence_status=candidate|evidence_checked|generalized|render_tested|audience_tested|active|rejected|deprecated`.

**TechniqueBundle:** query, selected_card_refs[], relevance_reasons[], capability_profile_ref, excluded_cards_with_reasons[]. Story и Cinematography KB могут использовать одно физическое хранилище, но разные kind/поиск и проверки. Ссылка на observation не заменяется пересказом модели.

## 5. CharacterBible, CanonSnapshot, LocationSpec и EpisodeBrief

| Тип | Обязательные поля |
|---|---|
| CharacterBible | characters[{character_id, original_name, skin_ref, rig_ref, silhouette_notes, movement_signature, traits, wants, weakness, expressive_capabilities[], prohibited_actions[], relationships[]}] |
| CanonSnapshot | series_id, accepted_episode_refs[], stable_facts[], current_relationships[], unresolved_threads[], canon_revision |
| CanonPatch | base_revision, proposed_fact_changes[], source_accepted_episode_ref; применяется только после приёмки |
| LocationSpec | location_id, world_snapshot_ref, coordinate_origin, walkable_regions[], blocked_regions[], camera_safe_regions[], anchors[], allowed_world_events[], lighting_profile |
| EpisodeBrief | duration_min_ms, duration_max_ms, characters[], locations[], main_conflict_limit, audience_assumptions[], no_dialogue, visual_style, complexity_budget, novelty_constraints[], required_capability_profile_ref |

`complexity_budget` задаёт max_characters=3, max_locations=2, max_simultaneous_actions, допустимые контакты, server event types и запрещённые эффекты. Для первого пилота предпочтительно ровно 2 героя и 1 локация.

## 6. StorySpec и StoryReview

**StorySpec:** brief_ref, character_bible_ref, canon_ref, logline, premise, main_conflict, emotional_payoff, estimated_duration_ms, beats[], causal_edges[], setups_and_payoffs[], required_capability_families[], originality_notes, technique_refs[].

**Beat:** beat_id, estimated_duration_ms, participating_characters[], character_goals[], obstacle, attempt, consequence, state_before[], state_after[], audience_should_understand, emotion_change[], depends_on_beat_ids[], necessity, criticality=`critical|supporting`.

`causal_edges={from_beat_id,to_beat_id,relation:enables|motivates|reveals|causes,explanation}`. Это причинный граф смысловых изменений, не просто порядок сцен. События могут составлять повествовательный цикл, но ссылки зависимостей на уже произошедшие beats не должны образовывать технический цикл исполнения.

**StoryReview:** story_ref, rubric_ref, decision=`pass|revise|reject`, dimension_scores[], defects[], required_changes[], estimate_of_remaining_complexity.

`Score={dimension, score:1..5|null, not_observable:boolean, evidence_refs[], rationale}`. Если not_observable=true, score=null.

Semantic checks: все ids существуют; суммарная длительность допустима; один основной конфликт; критический beat меняет состояние; payoff имеет setup либо обоснованное исключение; нет обязательной речи/надписей; нет действий вне CapabilityProfile; финал разрешает основной конфликт.

## 7. DirectionPlan: SceneSpec, PerformancePlan, ShotSpec

**DirectionPlan:** story_ref, capability_profile_ref, location_refs[], scenes[], performances[], shots[], edit_plan_ref, sound_plan_ref, beat_coverage[], continuity_constraints[].

**SceneSpec:** scene_id, beat_ids[], location_id, start_state_ref, end_state_predicates[], production_duration_ticks, participating_characters[], dramatic_function.

**PerformancePlan:** performance_id, scene_id, character_id, objective, emotion_before, emotion_after, actions[], visible_cues[], expected_reaction_to_event_ids[].

`Action={action_id, capability_id, capability_version, start_tick, end_tick, parameters, target_id|null, occupied_channels[], preconditions[], expected_postconditions[], execution_mode, causative_event_id|null}`.

Схема parameters определяется capability, а не свободным текстом. Занятость каналов берётся из registry, а не на доверии из LLM. `execution_mode=staged_visual|server_action`.

**ShotSpec:** shot_id, scene_id, beat_ids[], performance_refs[], source_start_tick, source_end_tick, handles_ticks, audience_inference, character_intentions[], readable_emotions[], camera, staging, reveal_schedule[], cut_rule, sound_cue_ids[], acceptance_assertions[], fallback_shot|null.

| Группа ShotSpec | Поля |
|---|---|
| camera | size=wide/medium/close/insert, position/path, target/path, yaw/pitch/roll или вычисляемое look_at, fov_deg, interpolation, movement_reason |
| staging | actor_marks[], attention_target_ids[], screen_side_by_character, interaction_axis, minimum_subject_height_ratio, protected_objects[], permitted_occlusions[] |
| reveal_schedule | event_id, visible_from_tick, visible_until_tick, purpose |
| cut_rule | cut_at_tick, motivation, required_action_or_reaction_id, incoming/outgoing screen direction |
| acceptance_assertions | predicate id, target id, interval, expected value/tolerance, severity, validation_method |

`beat_coverage={beat_id, shot_ids[], visible_evidence_description, minimum_read_time_ticks, critical_assertion_ids[]}`. Каждый critical beat должен иметь покрытие; «текст сценария говорит, что зритель понял» покрытием не считается.

### Иллюстрация цепочки от смысла до съёмки

Условный beat: A хочет, чтобы B последовал за ним; B видит, что A остановился и проверяет его реакцию. Без предметов и world events; проверяются движение, взгляд и пауза.

| Уровень | Описание |
|---|---|
| Story | A прекращает торопиться и впервые ждёт B; меняется его поведение по отношению к напарнику |
| Director | Общий статичный план показывает дистанцию; A останавливается и оглядывается; отдельная пауза; B отвечает кивком; A начинает движение только после реакции |
| Performance | move_path A → turn/look A → hold → nod B → move_path A/B; look и root разделены |
| Compiler | Разрешённые траектории и треки поворотов с допустимыми параметрами, без новой логики истории |
| Проверка | До кивка B A не продолжает путь; оба видны; поворот/кивок читается; слепой зритель связывает ожидание с реакцией |

Расписание 8-секундного фрагмента, 160 тиков, 240 выходных кадров при 30 fps:

| Тики | Действие | Что должно считываться |
|---|---|---|
| [0,40) | A идёт до отметки; B стоит позади | Между героями есть дистанция |
| [40,64) | A останавливается и поворачивается к B | A обратил внимание на B |
| [64,84) | Оба держат позу; A смотрит на B | A ждёт ответа |
| [84,104) | B кивает | Ответ B следует за ожиданием |
| [104,120) | Пауза после реакции | Зритель успевает связать действия |
| [120,160) | A и B начинают движение по одному направлению | A продолжает путь вместе с B |

Это демонстрация контракта, не утверждение, что все primitives уже реализованы или что фрагмент сам по себе является интересной историей. Успех выразительности проверяется в Minecraft. Компилятор не должен превращать «кивок» в случайную амплитуду без тестированного шаблона.

## 8. CapabilityProfile и PreflightReport

**CapabilityProfile:** backend_id/version/commit, minecraft_version, fabric_version, rig_versions[], render_profile_refs[], capabilities[], world_event_types[], restrictions[], evidence_refs[].

**Capability:** capability_id, version, parameter_schema_ref, execution_mode, preconditions[], postconditions[], channel_locks[], duration_range_ticks, safe_time_scale_range, asset_dependencies[], validated_camera_ranges[], implementation_status, fallback_ids[], render_evidence_refs[].

**PreflightReport:** direction_ref, capability_profile_ref, status=`pass|fail`, checks[], diagnostics[], compiled_take_refs[].

`Check={check_id, method, input_refs[], status, measured_value|null, threshold|null, evidence_ref|null}`.

Проверки включают: схемы/ссылки, интервалы, конечность координат, диапазоны, каналы, опору и коллизии, камеры в стенах, проекционный размер героя/предмета, occlusion, линии взглядов, continuity, длительность и доступность звука. Точные проверки коллизий/проекции — код; смысл улыбки/сочувствия — preview/зрители.

`pass` недопустим, если critical action имеет status=idea или не прошёл обязательный тест текущего backend/rig. Нельзя «починить» preflight тихим удалением ключевого действия: требуется новая версия DirectionPlan.

## 9. CompiledTake, RenderRequest, ExecutionTrace, TakeManifest

| Тип | Обязательные поля |
|---|---|
| CompiledTake | shot_ref, compiler_version, backend_profile_ref, world_snapshot_ref, start_state, pre_roll_ticks, duration_ticks, actor_tracks[], camera_tracks[], world_events[], end_state_assertions[], schedule_hash |
| WorldEvent | event_id, execute_at_tick, type, target, typed_parameters, required_server_ack, expected_world_state, idempotency_key |
| RenderRequest | job_id, take_id, compiled_take_ref, render_profile_ref, request_hash, output_artifact_namespace, timeout_ms, budget_grant_ref, attempt_number |
| ExecutionTrace | take_id, input_hashes, worker_version, tick/frame map, applied_events[], failed_assertions[], final_state, warnings |
| AppliedEvent | event_id, requested_tick, applied_tick, server_ack_tick|null, observed_state, status, failure_code|null |
| TakeManifest | take_id, shot_ref, render_request_ref, video AssetRef, trace_ref, actual_frame_count, actual_duration, fps, width/height, pixel_format, capture_time_map_ref, technical_qc_ref, start/end_state_refs |

RenderProfile задаёт размер, fps, codec, pixel format, quality, warmup policy и handles. Все пути разрешает worker в собственном namespace; модель не управляет путями или аргументами shell.

Состояния render job:

`queued → claimed → loading → resetting → warmup → recording → verifying → succeeded`.

Терминальные альтернативы: `failed`, `cancelled`, `timed_out`. Повтор доставки с тем же job_id и тем же request_hash возвращает результат; с другим hash — `IDEMPOTENCY_CONFLICT`. Результат succeeded публикуется только после QC; частичный файл не регистрируется как готовый asset.

Очередь MVP — at-least-once delivery + идемпотентность. Обещание exactly-once для процессов и мировых побочных эффектов не делается. Reset/snapshot и подтверждённые event ids обеспечивают безопасное восстановление.

## 10. EditPlan, SoundPlan и MasterVideo

**EditPlan:** output_profile, clips[{clip_id, shot_id, selected_take_id, source_in_frame, source_out_frame, timeline_in_frame, transition}], edit_constraints[], total_frames. До съёмки selected_take_id=null; Editor разрешает его в принятую версию take и сохраняет resolved EditManifest.

**SoundPlan:** sample_rate, cues[{cue_id, event_id|null, asset_id, scene_tick|null, timeline_frame|null, offset_samples, duration_samples, gain_db, pan, layer, semantic_purpose}], music_policy, loudness_target, peak_limit.

Каждый cue должен иметь ровно одну базовую привязку — scene_tick или timeline_frame; после монтажа всё разрешается в final sample_index. Если событие сдвинуто склейкой, его звук отображается через EditManifest, а не старый wall clock.

**MasterVideo:** video AssetRef, resolved_edit_manifest_ref, audio_mix_report_ref, technical_qc_ref, duration_frames, source_take_refs[], build_fingerprint.

`EditManifest` хранит соответствие final frame/time → clip → take source frame → shot/scene tick → beat/event. Это необходимо для конкретной локализации дефекта в готовом видео.

## 11. VideoReview, Defect, RepairProposal

**VideoReview:** master_ref, rubric_ref, analysis_sampling_ref, blind_reconstruction, intent_comparison, dimension_scores[], defects[], decision, review_limitations[].

`blind_reconstruction` создаётся до загрузки StorySpec: observed_goals[], inferred_causal_edges[], inferred_resolution, uncertain_moments[]. Результат фиксируется и затем не переписывается под сценарий.

**Defect:** defect_id, category, severity=`blocker|major|minor`, final_range_frames, shot_ids[], beat_ids[], observation, evidence_refs[], suspected_causes[{cause,module,confidence}], expected_behavior, suggested_fix, verification_method.

Категории: narrative_clarity, causality, expression, animation, camera, editing, pacing, engagement, story_fidelity, technical, audio. Обязательные девять художественных измерений оцениваются отдельно; technical/audio дополняют их.

**RepairProposal:** defect_refs[], target_module, target_artifact_ref, proposed_changes[], preserved_invariants[], dependency_invalidation_refs[], estimated_api_cost_usd, estimated_compute_minutes, expected_test, stop_condition.

Перед применением код проверяет scope и остаток бюджета. `proposed_changes` — ограниченные поля новой версии плана, не произвольный Python/Java. При низкой уверенности допускается дополнительный анализ вместо изменения.

## 12. ProductionManifest, бюджет и ошибки

**ProductionRequest:** brief_ref, input_reference_refs[], character_bible_ref, capability_profile_ref, budget_grant_ref|null, limits, requested_stages[].

**ProductionManifest:** production_id, artifact_graph, selected_versions, job_states, best_master_ref|null, unresolved_defects[], attempt_counters, cost_ledger_ref, approval_state, final_status.

`final_status=draft|running|needs_human_review|accepted|rejected|budget_exhausted|failed`. `accepted` требует художественной и человеческой приёмки по MVP-рубрике. Recorder.succeeded не переводит производство в accepted.

**BudgetGrant:** grant_id, explicit_user_authorization_ref, currency=USD, max_total, allocations_by_stage, max_cost_per_call, allowed_models[], allowed_endpoints[], allowed_stages[], expiry|null. При отсутствии разрешения или нулевом лимите допускаются только локальные операции без платного API.

**CostLedgerEntry:** request_id, stage, model, pricing_version/date, estimated_upper_cost, reserved_cost, actual_usage, actual_cost|null, state=`reserved|settled|unknown`, retry_of|null. Неизвестная стоимость не обнуляется после timeout.

**ModelInvocation:** role, requested_model, returned_model, snapshot|null, endpoint, prompt_version/hash, schema_version, input_asset_refs, input/output_usage, reasoning_usage|null, request_id, latency_ms, cost_ref, completion_status. Полный секретный credential в artifact не попадает.

Общий результат операции: `{status:ok|error, output_refs:[], error:null|Error}`.

`Error={code, message, stage, artifact_refs[], retryable, retry_after_ms|null, evidence_refs[], suggested_owner}`.

| Код | Поведение |
|---|---|
| INPUT_INVALID / SOURCE_UNREADABLE | Локально исправить вход; платный анализ не запускать |
| MODEL_REFUSAL / MODEL_INCOMPLETE / MODEL_OUTPUT_INVALID | Сохранить причину; ограниченный повтор только по policy и бюджету |
| UNSUPPORTED_CAPABILITY / ACTION_CHANNEL_CONFLICT | Вернуть Director с доступными альтернативами |
| CAMERA_OCCLUDED / VISIBILITY_TOO_LOW | Новая постановка, затем preflight |
| WORLD_STATE_MISMATCH / SERVER_EVENT_UNACKNOWLEDGED | Остановить дубль, reset; не публиковать как успешный |
| ENCODER_FAILED / VIDEO_QC_FAILED | Изолировать частичный файл; новый take при разрешённом retry |
| IDEMPOTENCY_CONFLICT | Отклонить конфликтующий запрос без перезаписи результата |
| STALE_DEPENDENCY | Пересчитать зависимые артефакты |
| BUDGET_NOT_AUTHORIZED / BUDGET_EXHAUSTED | Не отправлять платный запрос; сохранить результат и остаток работы |
| HUMAN_REVIEW_REQUIRED | Завершить автоматические изменения, предоставить видео/отчёт |

Изменение upstream-артефакта инвалидирует downstream по графу зависимостей. Кэш по типу, payload hash, версиям моделей/prompts/compiler/assets/backend, а не по похожему названию файла.
