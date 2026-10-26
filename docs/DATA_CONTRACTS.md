# Спецификация данных между модулями, версия 1.0

Дата: 26 октября 2026. Это нормативное проектное описание и иллюстрации; исполняемые схемы, API-клиенты и компилятор ещё не реализованы.

Стратегия: **Reference-Driven Iterative Film Production**. Версия 1.0 заменяет проектный контракт 0.1: добавлены обязательные mode/benchmark/experiment links и разделение гипотезы от результата. Это несовместимое расширение проекта, поэтому изменён major. Исполняемых артефактов 0.1 пока нет; при будущем импорте черновиков mode, benchmark и provenance задаются явно, а не угадываются. Исходные инженерные сущности сохранены.

Первый цикл использует только необходимое подмножество контрактов: два источника → benchmark → адаптация/постановка → V0 → alignment/comparison/gap → experiment → V1 → before/after и журнал участия человека. Большие библиотеки, канон и автоматический orchestration не обязательны для этого опыта.

## 1. Общие правила

Обмен — UTF-8 JSON. Все поля, перечисленные как обязательные, присутствуют; отсутствие знания выражается `null` или явным `unknown`, а не выдуманным значением. В будущей реализации — JSON Schema с закрытыми объектами и отдельные semantic validators. Совместимые добавления увеличивают minor schema version, изменение смысла/единиц — major.

Все выходные артефакты используют оболочку:

| Поле | Тип | Требование |
|---|---|---|
| schema_version | string | Для новой спецификации `mafs/1.0` |
| artifact_type | enum/string | Один из описанных ниже типов |
| artifact_id | string | Стабильный UUID или локальный уникальный id |
| revision | integer ≥ 1 | Новая редакция не меняет предыдущую |
| production_id | string | Общий id производства |
| parent_refs | ArtifactRef[] | Точные версии входных артефактов; для корня пустой массив |
| created_at | string | ISO-8601 UTC |
| producer | object | module, kind=`code|model|human`, code_revision (null для ручного результата); для AI также ModelInvocationRef |
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

**ReferenceAsset:** source_id, source_kind=`local_mp4`, private AssetRef, origin_url|null, creator_label|null, reference_role=`storytelling|cinematography|studio_baseline`, rights_record, duration_us, streams[], original_time_base, proxy_map_ref|null. Собственный принятый MasterVideo может стать studio_baseline с исходным master_ref и неизменным hash; происхождение не теряется.

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

Карточка также содержит experiment_refs[], before_after_evaluation_refs[], scope_of_validity, outcome_summary и retrieval_policy=`generalized_original_safe|internal_reconstruction_only`. Успешные, неудачные и inconclusive случаи сохраняются отдельно. Одна улучшенная сцена не переводит приём автоматически в универсально подтверждённый. Детальные реконструкции не попадают в original-safe retrieval.

## 5. CharacterBible, CanonSnapshot, LocationSpec и EpisodeBrief

| Тип | Обязательные поля |
|---|---|
| CharacterBible | characters[{character_id, original_name, skin_ref, rig_ref, silhouette_notes, movement_signature, traits, wants, weakness, expressive_capabilities[], prohibited_actions[], relationships[]}] |
| CanonSnapshot | series_id, accepted_episode_refs[], stable_facts[], current_relationships[], unresolved_threads[], canon_revision |
| CanonPatch | base_revision, proposed_fact_changes[], source_accepted_episode_ref; применяется только после приёмки |
| LocationSpec | location_id, world_snapshot_ref, coordinate_origin, walkable_regions[], blocked_regions[], camera_safe_regions[], anchors[], allowed_world_events[], lighting_profile |
| EpisodeBrief | production_mode, benchmark_case_refs[], duration_min_ms, duration_max_ms, characters[], locations[], main_conflict_limit, audience_assumptions[], no_dialogue, visual_style, complexity_budget, novelty_constraints[], required_capability_profile_ref |

`complexity_budget` задаёт max_characters=3, max_locations=2, max_simultaneous_actions, допустимые контакты, server event types и запрещённые эффекты. Для первого пилота предпочтительно ровно 2 героя и 1 локация.

`ProductionMode=reference_reconstruction|original_generation`. Первый цикл: reference_reconstruction, 30000–60000 ms; original_generation на 60000–120000 ms начинается после принятого сравнительного пилота. Это не те же enum, что execution_mode.

## 6. StorySpec и StoryReview

**StorySpec:** production_mode, benchmark_case_refs[], adaptation_record_ref|null, release_policy, brief_ref, character_bible_ref, canon_ref|null, logline, premise, main_conflict, emotional_payoff, estimated_duration_ms, beats[], causal_edges[], setups_and_payoffs[], required_capability_families[], originality_notes, technique_refs[]. Для reconstruction adaptation_record_ref обязателен, release_policy=internal_research_only, canon_ref может быть null; собственные герои стабильны и без канона сериала.

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

**RepairProposal:** defect_refs[], quality_gap_refs[], improvement_experiment_ref|null, target_module, target_artifact_ref, proposed_changes[], preserved_invariants[], dependency_invalidation_refs[], estimated_api_cost_usd, estimated_compute_minutes, expected_test, stop_condition. Для сравнительного исправления experiment_ref обязателен до применения; обычный технический retry не считается доказательством улучшения повествования.

Перед применением код проверяет scope и остаток бюджета. `proposed_changes` — ограниченные поля новой версии плана, не произвольный Python/Java. При низкой уверенности допускается дополнительный анализ вместо изменения.

## 12. ProductionManifest, бюджет и ошибки

**ProductionRequest:** production_mode, benchmark_case_refs[], brief_ref, input_reference_refs[], character_bible_ref, capability_profile_ref, budget_grant_ref|null, limits, requested_stages[].

**ProductionManifest:** production_id, production_mode, benchmark_case_refs[], comparison_report_refs[], experiment_refs[], before_after_evaluation_refs[], manual_intervention_refs[], automation_scope, release_policy, artifact_graph, selected_versions, job_states, best_master_ref|null, unresolved_defects[], attempt_counters, cost_ledger_ref, approval_state, final_status.

`automation_scope=manual_assisted|hybrid|autonomous` с перечнем ручных/автоматических стадий. Reconstruction всегда internal_research_only и не меняет публичный CanonSnapshot; даже accepted означает исследовательскую приёмку, а не право выпустить его как оригинальный фильм.

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
| BENCHMARK_NOT_FROZEN / BENCHMARK_CHANGED | Не объявлять before/after валидным; заморозить новый case/эксперимент |
| ALIGNMENT_INSUFFICIENT / EVIDENCE_INSUFFICIENT | Дополнительный просмотр либо not_comparable; не присваивать нулевой балл |
| EXPERIMENT_PROTOCOL_VIOLATION | Сохранить результат как invalid; не подтверждать гипотезу |
| RECONSTRUCTION_RELEASE_BLOCKED | Не включать внутреннюю реконструкцию/её детальный skeleton в публичный оригинальный выпуск |

Изменение upstream-артефакта инвалидирует downstream по графу зависимостей. Кэш по типу, payload hash, версиям моделей/prompts/compiler/assets/backend, а не по похожему названию файла.

## 13. Общие сравнительные сущности и наблюдения фильма

Ниже перечисленные поля обязательны; nullable помечены явно, массивы могут быть пустыми, если это не запрещено инвариантом. Все сущности используют оболочку раздела 1 и ArtifactRef с revision/hash. Значения гипотез, прогнозов и измерений — разные типы и разные артефакты.

`NarrativeDimension=semantic_structure|goal_clarity|causality|action_readability|emotional_reaction|key_moment_staging|pacing|editing|payoff_effectiveness`.

Это сравнительная рубрика из девяти измерений. Она дополняет, а не переименовывает девять QC-измерений VideoReview: например, animation и story_fidelity остаются там. RubricRef хранит явное соответствие между рубриками; их средние не смешиваются.

`MediaEvidenceRef={source_ref, source_sha256, timeline:source_pts|master_frames, start_us, end_us, frame_range|null, evidence_artifact_refs[], coverage, gaps_us[]}`. Source_ref указывает ReferenceAsset либо MasterVideo. Для source_pts сохраняются PTS/time_base в исходном Evidence; для master_frames frame_range обязателен и переводится через fps/EditManifest. Интервалы полуоткрытые, в пределах источника. Скриншот подтверждает позу, но не весь процесс действия.

**ObservedNarrativeEvent:** event_id, source_ref, evidence_refs[] (непустой), observed_description, interpreted_function|null, actor_local_ids[], visible_goal_cues[], causal_hypothesis_refs[], emotion_cues[], interpretation_confidence|null, alternatives[], review_status=`unreviewed|confirmed|disputed|uncertain`. Причинные связи остаются CausalHypothesis с evidence/confidence; видимая последовательность сама по себе не доказывает причину.

**FilmAnalysis:** master_ref, analysis_policy_ref, blind_review_ref, observed_events[], causal_hypotheses[], inferred_resolution|null, uncertain_intervals[], coverage_report_ref. Анализ должен быть сформирован до доступа к StorySpec и предполагаемому победителю. Ссылка на story beat допустима только в последующем intent-comparison pass, не в observed evidence.

**AdaptationRecord:** production_mode=reference_reconstruction, benchmark_case_ref, source_structure_ref, adapted_story_id, mappings[{source_event_ids[], story_beat_ids[], preserved_function, modification, reason, capability_constraint_refs[], meaning_changed:boolean}], omitted_events_with_reasons[], preserved_causal_invariants[], reviewer_decisions[], release_policy=internal_research_only.

Modification может быть none; остальные изменения не замалчиваются. Adapted_story_id и story_beat_ids — заранее выделенные логические ids; затем StorySpec ссылается на замороженный AdaptationRecord. Обратного hash-ref на эту же StorySpec нет, чтобы не создать циклические content hashes. Semantic invariant проверяет причинность, не сходство чужих персонажей. Source skeleton и детальная adaptation остаются приватными исследовательскими артефактами.

## 14. ReferenceBenchmarkCase

| Поле | Тип / требование |
|---|---|
| benchmark_case_id / production_mode | string / ProductionMode |
| research_question / target_audience | string / описание предпосылок зрителя |
| sources | Массив BenchmarkSource; в первом опыте ровно один Alan/storytelling и один Ethobot/cinematography |
| source_analysis_refs / source_structure_ref | ArtifactRef[] / ArtifactRef|null; structure обязателен для reconstruction |
| relevant_functions / critical_reference_event_ids | string[] / string[] |
| rubric_ref / metric_protocol_ref | ArtifactRef / ArtifactRef |
| capability_profile_ref / allowed_adaptations | ArtifactRef / структурированные ограничения и инварианты |
| selection_rationale / exclusions | string / список несравнимых аспектов и причин |
| prior_studio_baseline_refs | ArtifactRef[]; пустой в первом опыте |
| freeze_state / frozen_at | draft или frozen / UTC|null |
| human_review_refs / release_policy | ArtifactRef[] / internal_research_only или generalized_original_safe |

`BenchmarkSource={source_ref, source_sha256, selected_ranges_us[], role, applicable_dimensions[], applicable_event_ids[], relevance_rationale, limitations[], rights_record_id}`. Роль и применимые измерения назначаются до baseline. Ethobot не становится драматургическим эталоном по умолчанию, Alan не доказывает доступность Minecraft-анимации. В Original Mode источники могут быть другими и включать собственные фильмы; обязательность двух конкретных авторов относится только к первому опыту.

Замороженный case неизменяем по revision/hash. Изменение диапазона, критического события, рубрики, аудитории или области применимости требует новой revision и нового experiment. Pixel/style similarity не может быть primary metric_protocol. Референсные и собственные таймкоды не обязаны совпадать; case не задаёт одинаковые кадры.

## 15. NarrativeBeatAlignment

**Поля:** alignment_id, production_mode, benchmark_case_ref, film_analysis_ref, master_ref, reference_analysis_refs[], links[], unmatched_reference_events[], unmatched_film_events[], coverage_summary, method, reviewer_refs[], limitations[].

`AlignmentLink={link_id, reference_source_ref, reference_event_ids[], reference_evidence_refs[], film_event_ids[], film_evidence_refs[], semantic_function, relation:one_to_one|one_to_many|many_to_one|many_to_many|unmatched, status:comparable|partial|not_comparable|insufficient_evidence, comparability_rationale, adaptation_record_ref|null, capability_constraints[], alignment_confidence, alternative_matches[], review_status}`.

Для comparable обе стороны непустые и с наблюдаемым evidence. Для unmatched отсутствующая сторона пуста; известная сторона, предполагаемая функция, проверенный диапазон другой стороны и степень coverage обязательны в `UnmatchedEvent={event_id, side, reason, expected_function, searched_ranges[], evidence_refs[], coverage, review_status}`. Редкая выборка не доказывает отсутствие события. Отрицательное свидетельство отсутствия требует плотного просмотра релевантного диапазона.

Coverage_summary перечисляет все critical events с итоговым link/unmatched/exempt и знаменателями. Нельзя незаметно исключить неудачный beat из оценки. Многие планы могут выражать один beat, один план — несколько; никакой минимизации пиксельной дистанции или принудительной 1:1-синхронизации.

## 16. ReferenceComparisonReport

**Поля:** comparison_id, production_mode, benchmark_case_ref, master_ref, film_analysis_ref, alignment_ref, rubric_ref, comparison_policy_ref, dimension_findings[], quality_gap_ids[], strengths[], not_comparable_items[], insufficient_evidence_items[], critical_event_coverage, human_review_refs[], evaluator_provenance, limitations[].

`DimensionFinding={dimension:NarrativeDimension, alignment_link_ids[], reference_observations[], film_observations[], reference_evidence_refs[], film_evidence_refs[], descriptive_metrics[], interpretation, assessment:gap|parity|strength|not_comparable|insufficient_evidence, confidence, human_review_status}`.

`descriptive_metrics` содержат id, value, unit, method, interval и source; их значение не становится автоматически оценкой качества. Опциональные ordinal scores разрешены только с рубрикой и rationale; единый средний score не является решением приёмки. Baseline/candidate оцениваются одинаковым протоколом; comparator не получает identity победителя или ожидаемую пользу правки в первом анализе.

Все findings должны иметь link ids и evidence соответствующей стороны. not_comparable и insufficient_evidence не записываются как ноль и сохраняются в coverage. Отчёт об отличии не является BeforeAfterEvaluation и не доказывает, что предложенная правка полезна.

Quality_gap_ids — выделенные логические ids; точные версии QualityGap разрешаются через manifest. Сам QualityGap ссылается на hash готового отчёта. Отчёт не включает обратный hash будущего gap, поэтому зависимость остаётся ациклической.

## 17. QualityGap

**Поля:** gap_id, comparison_report_ref, dimension, alignment_link_ids[], severity=`major|minor`, observed_event_refs[], reference_evidence_refs[], film_evidence_refs[], observed_difference, audience_effect_hypothesis, suspected_causes[], alternative_explanations[], adaptation_or_capability_constraints[], proposed_fix, verification_plan, reviewer_decisions[], status=`proposed|confirmed|disputed|resolved|not_supported`.

`suspected_causes=[{layer:story|adaptation|direction|execution|capability|editing|audio|analysis_sampling|unknown, hypothesis, confidence, supporting_evidence_refs[], contradicting_evidence_refs[]}]`.

`proposed_fix={target_module, target_artifact_refs[], change_description, feasible_with_capabilities[], fallback|null}`.

`verification_plan={observable_prediction, metric_ids[], evidence_to_collect[], human_question_ids[], guardrails[], proposed_minimum_effect}`. В ImprovementExperiment предсказание и пороги замораживаются до вмешательства.

Недостаток обязан ссылаться на наблюдаемое событие либо документированное отсутствие ожидаемого события с достаточным coverage, содержать гипотезу причины, способ исправления и проверки. «Не похож на скриншот референса» или «AI предпочёл» не является достаточным gap. Major — нарушение считывания критической цели/причины/реакции/payoff; два рецензента подтверждают первый major-gap. Их решения не превращают причинную гипотезу в факт.

## 18. ImprovementExperiment

| Поле | Тип / назначение |
|---|---|
| experiment_id / production_mode | string / ProductionMode |
| benchmark_case_ref / baseline_master_ref | Точные замороженные ArtifactRef |
| baseline_comparison_ref / target_gap_refs | ArtifactRef / непустой ArtifactRef[] |
| hypothesis / observable_predictions | Причинное предположение / ожидаемые проверяемые изменения |
| intervention_plan | target artifacts, factor/bundle, patch description, executor kind, preserved invariants |
| controlled_variables / known_confounders | Что сохраняется / что нельзя контролировать |
| primary_metric / minimum_effect | MetricSpec / величина и единица; до вмешательства |
| guardrail_metrics / evaluation_protocol_ref | MetricSpec[] / HumanEvaluationProtocol + автоматические проверки |
| allowed_attempts / budget_grant_ref | integer (1 candidate по умолчанию) / ArtifactRef|null |
| preregistered_at / registration_state | UTC|null / draft или registered |

`MetricSpec={metric_id, construct, source:code|model|human, operational_definition, question_or_check_ref, unit, aggregation, direction:higher_better|lower_better|target_range, target_range|null, sampling_unit, uncertainty_method}`.

После registered артефакт неизменяем; корректировка гипотезы/метрик/benchmark создаёт новый experiment. Runtime state proposed/executed/evaluated/stopped, реальные intervention/candidate refs, deviations и итоговый evaluation_ref хранятся в manifest/журнале опыта, а не дописываются в замороженный протокол. Так BeforeAfterEvaluation ссылается на experiment без обратной циклической hash-зависимости. Реальные изменения в отдельном InterventionRecord не могут быть выданы за первоначальный план. Один фактор предпочтителен; bundle должен называться bundle. Все candidate-варианты и rejected эксперименты сохраняются, нельзя выбирать только успешный и скрывать остальные.

**InterventionRecord:** intervention_id, experiment_ref, input_artifact_refs[], output_artifact_refs[], actual_changes[], executor=`human|code|model_assisted_human|automated`, manual_intervention_refs[], started_at, finished_at, actual_cost_ref|null, deviations_from_plan[], evidence_refs[].

**ManualInterventionRecord:** intervention_id, stage, experiment_ref|null, actor_role, input_refs[], output_refs[], tool_and_version, reason, actual_changes[], active_minutes, decision_origin=`human|model|mixed`, model_proposal_ref|null. Запись может относиться к подготовке baseline; не требуется притворяться, что она была автоматической. Вклад человека и модели не смешивается.

## 19. BeforeAfterEvaluation

**Поля:** evaluation_id, experiment_ref, benchmark_case_ref, baseline_master_ref, candidate_master_ref, before_comparison_ref, after_comparison_ref, actual_intervention_refs[], protocol_ref, protocol_adherence, deviations[], metric_results[], guardrail_results[], human_results_ref, technical_qc_refs[], outcome, hypothesis_assessment, interpretation, limitations[], knowledge_updates[].

`MetricResult={metric_id, baseline_value, candidate_value, delta, unit, baseline_n, candidate_n, sampling_unit, raw_observation_refs[], uncertainty_interval|null, interval_method, meets_minimum_effect:boolean|null}`.

`outcome=improved|no_change|regressed|inconclusive|invalid`; `hypothesis_assessment=supported_in_this_case|not_supported|unresolved`. Улучшение bundle не подтверждает его отдельные компоненты. `improved` требует заранее принятого primary effect, человеческой проверки, технического прохода и всех critical guardrails. Нехватка/противоречивость evidence даёт inconclusive; нарушение протокола — invalid. Нельзя вывести improved только из model-score или изменения самого плана без фильма.

Сравнение V0/V1 с референсами и сравнение V0/V1 между собой — разные результаты; оба сохраняются. Референс не переоценивается под желаемого победителя. Failed/inconclusive не удаляются из knowledge history. Новое правило получает scope конкретного случая, пока нет независимого повторения.

Knowledge_updates внутри оценки — предложения правил/ограничений с логическими card ids, не hash-ссылки на будущие карточки. Применённые обновления регистрируются позже в manifest; новая TechniqueCard ссылается на готовую оценку. Граф evidence и content hashes остаётся ациклическим.

**HumanEvaluationProtocol:** protocol_id, preregistered_at, audience_criteria, participant_count, allocation_and_randomization, display_conditions, blind_labels, question_specs[], answer_key_ref, primary_metric_id, minimum_effect, guardrails[], preference_question, tie_policy, exclusion_rules, uncertainty_method, rater_procedure, stop_rule, reference_exposure_policy.

**HumanEvaluationResults:** protocol_ref, blinded_variant_hash_map_ref, anonymized_responses[{participant_id, first_variant, order, first_view_answers, ratings, paired_preference, prior_familiarity, exclusion_reason|null}], rater_decisions[], summary_counts, uncertainty_report, actual_n, protocol_deviations[]. Личные идентификаторы и ответы хранятся приватно; для публикации — только допустимые агрегаты.

Участник — единица независимого наблюдения; пять ответов одного зрителя не становятся пятью независимыми участниками. Ответы первого показа собираются до второго фильма, сценария и референса. Порог, правило ties/exclusions и объём группы фиксируются заранее; после просмотра их не меняют ради pass.

### Обязательные сравнительные инварианты

1. Одна цепь полностью разрешается: benchmark → события/evidence → alignment → report/gap → registered experiment → actual intervention → candidate → новый report → BeforeAfterEvaluation.
2. У всех звеньев совпадают production mode, baseline/candidate hashes и замороженный benchmark. Исключение — явно новый experiment; старая оценка не переписывается.
3. V0 и V1 — разные immutable версии MasterVideo, обе проходят TechnicalQC. Все ручные правки имеют записи, затраты и новую версию артефакта.
4. Никаких фиктивных film events из StorySpec; отсутствие не утверждается по неполной выборке. Confidence — оценка, не вероятность истинности.
5. Планируемая польза живёт в ImprovementExperiment; измеренный результат — только в BeforeAfterEvaluation. Null outcome до оценки не заменяется оптимистичным прогнозом.
6. Research accepted не означает autonomous production или public release. Original Mode не получает детальный skeleton реконструкции через KB/retrieval.
7. Первый пилот подтверждает локальный эффект; заявление о систематичности ссылается на независимые последующие experiment cases, а не на число прогонов одного фильма.
