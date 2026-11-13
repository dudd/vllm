# 研究发现：/v1/chat/completions 调用链

## 待填：源码锚点
（分析过程中记录关键文件与行号）

## HTTP 层链路（代理 1，已完成）
- build_app：api_server.py L157-314；中间件逆序执行：CORSMiddleware L256-262 → AuthenticationMiddleware（server_utils.py L38-86，sha256+compare_digest，仅 /v1 前缀，401 在 L84-85）→ XRequestIdMiddleware L277-280 → ScalingMiddleware L283（elastic_ep）→ 用户自定义 L301-311。异常处理器 L264-269（server_utils.py L321-445）。
- v1 路由：generate/api_router.py register_generate_api_routers（api_server.py L198-203 调用）→ chat_completion/api_router.py L105 attach_router；state.openai_serving_chat 在 generate/api_router.py L122-131（init_app_state 于 run_server L599-600）。
- validate_json_request：openai/utils.py L43-49。
- 装饰器：entrypoints/utils.py listen_for_disconnect L41-53（ASGI http.disconnect）；with_cancellation L56-98（asyncio.wait FIRST_COMPLETED，handler 返回 None→协程取消）；load_aware_call L105-150（计数 + BackgroundTask decrement L101-102）。
- create_chat_completion：api_router.py L53-74；分支 ErrorResponse L63-66 / ChatCompletionResponse L68-72 / StreamingResponse L74；metrics_header orca_metrics.py L100-120。
- OpenAIServingChat.create_chat_completion：serving.py L219-372；render_chat_request L192-217（→ entrypoints/serve/render/serving.py L185）；request_id L250-256；per-engine_input 循环 L266-345（n>1 子 id `_i` L273-275）；get_max_tokens entrypoints/utils.py L174-203；beam search 分支 L287-296（to_beam_search_params/to_sampling_params）；reasoning_ended L319-332；engine_client.generate L334-343；流式/非流式分支 L350-372。
- 流式生成器：serving.py L518-1256；num_choices L535；auto tool 判定 L1751-1765；主循环 L627-628；role chunk L637-656；_raise_if_error L1075-1077（基类 L385）；异常 L1249-1256 + [DONE] L1256。
- 非流式：serving.py L1258-1660；断连→CancelledError→"Client disconnected" L1277-1278；harmony 分支 L1314-1363；reasoning 拆分 L1365-1375；tool 提取 L1377+。
- 取消传播：async_llm.py generate L523-630；except CancelledError/GeneratorExit L588-593 → abort L706-715 → output_processor.abort_requests + engine_core.abort_requests_async。本版本不用 raw_request.is_disconnected 轮询。

## 协议/渲染层（代理 2）
- protocol.py：ChatCompletionRequest L150-828；校验器 L570-828（check_logprobs L609、check_structured_outputs_count L641、check_tool_usage L681）；to_sampling_params L469-568（_DEFAULT_SAMPLING_PARAMS L442）；to_beam_search_params L451-467（use_beam_search 字段 L191）；build_chat_params L404、build_tok_params L424。DeltaMessage 在 engine/protocol.py L258。
- chat_utils.py：MODALITY_PLACEHOLDERS_MAP L96；BaseMultiModalItemTracker L459（add L520）；parse_chat_messages L1600 / async L1636；_parse_chat_message_content L1510；_get_full_multimodal_text_prompt L1186；load_chat_template L1166。
- 渲染体系（新版）：OpenAIServingRender.render_chat（entrypoints/serve/render/serving.py L185）；preprocess_chat L520（tool_parser.adjust_request L594）；harmony 分支 _make_request_with_harmony L392-441；HfRenderer.render_messages_async（renderers/hf.py L672）；safe_apply_chat_template L439（tokenizer.apply_chat_template L464）。
- 工具解析：vllm/tool_parsers/（顶层，非 entrypoints）；ToolParser 基类 abstract_tool_parser.py L43（adjust_request L85、extract_tool_calls_streaming L138）；hermes L54（流式 L226、_partial_tag_overlap L33、args diff L214-224）；llama L39。ParserManager（vllm/parser/parser_manager.py L23，get_tool_parser L190）。
- Harmony：stream_harmony.py L28-171 extract_harmony_streaming_delta；harmony_utils.py（get_streamable_parser_for_assistant L331、parse_chat_output L342、render_for_completion L319）。
- use_harmony 判定：serving.py L163（model_type=="gpt_oss"）。

## AsyncLLM / IPC（代理 3）
- AsyncLLM.generate L523-632（消费循环 L572-583；异常分支 L588-632）；add_request L282-397（n>1 fan-out L384-397）；_add_request L399-414；output_handler L634-704（get_output_async L657、chunk L668-680、stop-string abort L683-686、propagate_error L700-702）；abort L706-718。
- InputProcessor.process_inputs L195-335（tokenize 委托 inputs/preprocess.py L274→renderers/base.py L380-391 encode；mm_features L286-319；EngineCoreRequest 构造 L321-335）；_validate_prompt_len L337-382（超长 L354-372）；assign_request_id L175-193。
- core_client.py：层级 EngineCoreClient→InprocClient(L274)/MPClient(L460)→SyncMPClient(L716)/AsyncMPClient(L887)/DPAsyncMPClient(L1137)/DPLBAsyncMPClient(L1317)。add_request_async L1058-1061（_send_input L1001-1011、send_multipart L1026）；输出 task L921-988（recv L945、decode L947、queue L979）；get_output_async L990-999；validate_alive L447-451。
- core.py：EngineCoreProc.run_engine_core L1060；input thread process_input_sockets L1368-1460（ADD decode L1443、preprocess_add_request L761-783：mm cache→Request.from_engine_core_request→grammar_init）；busy loop L1160-1168；_handle_client_request L1262-1295；EngineCore.step L404-442（schedule L415、execute_model L416、grammar bitmask L417-424、abort 队列 L428、update_from_output L429）；output thread L1462-1527（PUSH send L1517-1524）。
- IPC：client ROUTER/bind ↔ server DEALER/connect(identity 2B)；server PUSH ↔ client PULL；帧 [identity, type(1B), payload...]；EngineCoreRequestType（v1/engine/__init__.py L230-242：ADD b"\x00"、ABORT b"\x01"、UTILITY b"\x03"）；msgpack 在 v1/serial_utils.py（MsgpackEncoder L136，tensor 零拷贝 aux buffer，MultiModalKwargsItem L275-310）。
- parallel_sampling.py：ParentRequest L36-50（child params seed+index L52-81；child_req_id f"{index}_{request_id}" L83-94；get_outputs 聚合 L100-126）。OutputProcessor 侧 L321-327、L499-505。

## Scheduler / KV（代理 4）
- schedule() L348-954：token_budget L367；running 循环 L385-551（num_new_tokens L404-417；encoder 分支 L423-435 _try_schedule_encoder_inputs L1112-1287；allocate_slots L463→None 时抢占 L475-502；spec 分支 L520-536；encoder allocate L543）；waiting 循环 L563-855（prefix cache L613 get_computed_blocks、connector L619 get_num_new_matched_tokens、chunked prefill L673-689、allocate_slots L754、None→break L765-772、WAITING→new / PREEMPTED→resumed L821-824）；SchedulerOutput 构造 L919-935；_update_after_schedule L983-1007。
- 抢占：_preempt_request L961-981（free KV+encoder、num_computed_tokens=0、waiting 队首；v1 只有 recompute 模式）。
- KVCacheManager：get_computed_blocks L176-216（max_cache_hit_length=num_tokens-1 L201）；allocate_slots L257-427（KV 不足判定 L387-389；cache_blocks L413-425）；free L429-437。BlockPool：cache_full_blocks L211-320（hash 提交点）；get_new_blocks L322；_maybe_evict_cached_block L354-389（LRU）。
- encoder_cache_manager：can_allocate L119-178、allocate L180-205、free L242、get_freed_mm_hashes L255。
- abort：finish_requests L1759-1820；EngineCore aborts_queue L217，_process_aborts_queue L563-571（step 内 L428）。
- 完成释放：update_from_output L1299-1560（spec 验证 L1363-1387、encoder 释放 L1389-1391、stop 处理 L1400-1442、_free_request L1822-1838、_free_blocks L1840-1843）。
- spec decode：scheduler 钩子 L520-536、update_draft_token_ids L1673-1693；proposer 在 worker。
- KV connector：接口在 distributed/kv_transfer/kv_connector/v1/base.py（KVConnectorBase_V1 L170；get_num_new_matched_tokens L450、update_state_after_alloc L485、build_connector_meta L506、request_finished L530）；scheduler 调用点 L619/L779/L959/L2032。
- SchedulerOutput：v1/core/sched/output.py L179-253。
- beam search：不在 v1 scheduler；OpenAI 路径 serving.py L311-318 self.beam_search(...)（engine/serving.py L160+，客户端逐步扩展，每 beam 独立子请求，logprobs=2*beam_width）。

## Worker（代理 5）
- Worker.execute_model gpu_worker.py L755-844；GPUModelRunner.execute_model L3770-4122：_update_states L1059-1381→_prepare_inputs L1774-2084（block_table commit L1789、positions GPU L1990、input_ids 上传 L1999-2010、spec metadata L2027-2064）→_determine_batch_execution_and_padding L3536-3650→_get_slot_mappings L3687→_build_attention_metadata L2085→_preprocess L3193-3310（mm encoder L3208-3232）→_model_forward L3483-3514（forward 调用 L3510）→compute_logits L4085-4087→返回 None + ExecuteModelState L4112-4122。
- sample_tokens L4123-4345：grammar bitmask L4165、_sample L3311-3340（Sampler.forward sampler.py L68-144：greedy L240-251、temperature L255-257、topk_topp L265-270、penalties L357-392、logprobs L295-343；spec→RejectionSampler rejection_sampler.py L100）→propose_draft_token_ids L4491-4733（eagle propose eagle.py L404）→_bookkeeping_sync L3341-3467→ModelRunnerOutput L4301-4315（outputs.py L166-205）或 AsyncGPUModelRunnerOutput L4317-4345（异步 D2H）。
- CUDA graph：cudagraph_dispatcher.py dispatch L234-323（回退 eager L266-273；padding 表 _compute_bs_to_padded_graph_size L71-108）；capture sizes config/compilation.py L611/L655-667（默认 [1,2,4]+range(8,256,8)+range(256,max+1,16)，max=min(max_num_seqs*2,512)）；capture gpu_model_runner L6002-6093；replay cudagraph_utils.py run_fullgraph L247-262。
- KV connector mixin：kv_connector_model_runner_mixin.py maybe_get_kv_connector_output L58-69（start_load_kv L105、wait_for_save L111）；finalize_kv_connector L71-83。
- executor：multiproc_executor.py execute_model L306-335、collective_rpc L339-360。
- async scheduling：execute_model/sample_tokens 两段 RPC，无 AsyncSchedulerOutput 类。

## 输出处理/响应（代理 6）
- output_processor.py：process_outputs L572-687（stop string 检测 L632-637、logprobs L641、make_request_output L644、queue.put L657、reqs_to_abort L670-675）；RequestState L129-205、from_new_request L208-267（LogprobsProcessor L223 + IncrementalDetokenizer L227）、make_request_output L269-331（n>1 parent_req.get_outputs L324）；abort_requests L446-506（ABORT 输出 L493、父→子级联 L499-505）；propagate_error L439-445；do_tracing L706-765（OTEL）。
- detokenizer.py：工厂 L30-65；BaseIncrementalDetokenizer.update L95-142（stop string L131-140、stop_buffer_length L153）；FastIncrementalDetokenizer L167-242（DecodeStream）；SlowIncrementalDetokenizer L245-301（detokenize_incrementally）；check_stop_strings L304-339。
- logprobs.py：_update_sample_logprobs L69-119（cumulative L108）、_update_prompt_logprobs L121-187（UTF-8 修正 _correct_decoded_token L249、_verify_tokens L312）、pop_prompt_logprobs L189。
- 流式 serving.py：first chunk L637-679；delta_text L728-781（harmony L748-765 / 普通 L773）；previous 差分 L786-803；named tool 流式 L842-922；required L926-982（_filter_delta_text L393-416）；parse_delta L984-992；finish chunk L1074-1173（工具参数尾差补偿 L1092-1143：prev_tool_call_arr vs streamed_args_for_tool → remaining_call L1139，_should_check_for_unstreamed_tool_arg_tokens L1767-1789、_create_remaining_args_delta L1792-1818）；usage chunk L1198-1221；[DONE] L1256。
- 非流式 serving.py L1258-1661：断连 L1277-1278；harmony L1314-1363（parse_chat_output）；reasoning L1365-1375；tool 分支 L1392-1554；usage L1597-1612；clamp_prompt_logprobs L1622（engine/serving.py L749-762，-inf→-9999.0）；ChatCompletionResponse L1616-1627。
- 异常映射：engine_error_handler server_utils.py L321-361（terminate_if_errored launcher.py L156-166）；create_error_response entrypoints/utils.py L302-358（ValueError/TypeError→400、NotImplementedError→501、其余→500）。
- abort 链：with_cancellation→generate except L588-593→abort L706-718→output_processor.abort_requests L446-506→finish_reason "abort"。

## 启动链（代理 7）
- cli/main.py L15 main → cli/serve.py L122 uvloop.run(run_server) → api_server.py L671 run_server → L534 setup_server（bind 端口 L555-559、ulimit、signal）→ L681 run_server_worker → L78 build_async_engine_client → L109 from_engine_args（create_engine_config arg_utils.py L1567）→ AsyncLLM.from_vllm_config（async_llm.py L205）。
- AsyncLLM.__init__ L73：renderer_from_config（renderers/registry.py L80→tokenizers/registry.py L195 get_tokenizer lru_cache L261）L152；InputProcessor L158；OutputProcessor L162；make_async_mp_client L167-177；StatLoggerManager L180；output_handler task L195-176。
- MPClient.__init__ core_client.py L473：zmq ctx L522、launch_core_engines（utils.py L955：CoreEngineProcManager L83→Process(run_engine_core) L118-126、wait_for_engine_startup L1106）L546-549；READY 轮询 L608-627；monitor L641。
- EngineCoreProc.run_engine_core core.py L1060 → __init__ L809（handshake L825-830、EngineCore init L855、io threads L867-887）。
- EngineCore.__init__ L92-228：executor_class L109（MultiprocExecutor._init_executor multiproc_executor.py L109：rpc_broadcast_mq L135、WorkerProc.make_worker_process L157-190）；_initialize_kv_caches L230-312（get_kv_cache_spec L237、determine_available_memory L251、get_kv_cache_configs L272、initialize_from_config L293）；StructuredOutputManager L127；Scheduler L131-146。
- WorkerProc L578：init_worker→Worker.init_device gpu_worker.py L219（NCCL L268、MemorySnapshot L285、GPUModelRunner L300-320）→load_model L318（gpu_model_runner L4747）→determine_available_memory L332（profile_run L5754、cudagraph mem L393-405）→initialize_from_config L519（initialize_kv_cache L6738）→compile_or_warm_up_model L554（dummy_run L578、capture_model L6002）。
- build_and_serve api_server.py L578：get_supported_tasks L592、build_app L594（L157-314）、init_app_state L317（models L370、render L377、tokenization L392、init_generate_state→OpenAIServingChat generate/api_router.py L123、warmup L130-132）→serve_http（launcher.py L26，uvicorn L71-81，watchdog L80）。
- OpenAIServingChat.__init__ serving.py L91（ParserManager L131-139、default_sampling_params L163-172、harmony L173-181）；warmup L183。
- /health：serve/instrumentator/health.py L22→AsyncLLM.check_health async_llm.py L868。ready = engine READY 握手 + init_app_state + uvicorn startup。

## 分支补充（代理 8）
- harmony 流式：serving.py L540-544（每 choice StreamableParser）、L748-770 process(token_id)、L805-814 extract_harmony_streaming_delta、L1153 harmony_tools_streamed；非流式 L1314 parse_chat_output。
- beam search：protocol.py L191/L451-467；serving.py L287-289/L311-318；engine/serving.py L160 beam_search（逐 token 步循环，每 beam 子请求）；batch 接口拒绝 L841/L866/L888-891。
- prompt logprobs：clamp_prompt_logprobs engine/serving.py L749-762；校验 protocol.py L611-641；max_logprobs config/model.py L216（20）；SamplingParams 校验 L653-677。
- 引擎死：core_client validate_alive L447-451、_format_exception L621-628、monitor L655-661；SyncMPClient decode 异常 L762-774；async put EngineDeadError L981-984；abort 时 engine_dead 不发送 L829。
- Prometheus：metrics/loggers.py PrometheusStatLogger L404（num_requests_running L451、TTFT histogram L754、record L1058）；StatLoggerManager L1268。
- OTEL：tracing/otel.py instrument_otel L134、manual_instrument_otel L183；output_processor do_tracing L706-765；trace headers engine/serving.py L565-575。
- KV connector worker 侧：mixin L96 bind_connector_metadata / L102 start_load_kv / L78 wait_for_save；gpu_model_runner L3843 no_forward、L4030、L4300 finalize 延后。
- CUDA graph padding：dispatcher L71-105 padding 表、L234-320 dispatch 回退；compilation.py L611/L655-667；gpu_model_runner L3579 SP pad→L3581 dispatch→L3594 num_tokens_padded。
- mm encoder：_execute_mm_encoder L2715-2898（写 encoder_cache[mm_hash] L2893）；execute_model 内 prefill forward 前 L3214-3230；释放条件 scheduler.py L1665-1670。
