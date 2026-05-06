# 洛克家园功能增量补丁
# 适用于 pr0.4 分支，添加上游家园功能
# 生成时间: 2026-05-06

## 补丁说明

本补丁从上游 master 分支提取了洛克家园功能的核心代码，包括：
1. 家园查询命令 (/洛克家园)
2. 家园订阅命令 (/订阅家园菜园、/订阅家园灵感)
3. 取消订阅命令 (/取消订阅家园)
4. 相关辅助方法和数据处理逻辑

## 需要修改的文件

### 1. main.py

#### A. 在 import 部分添加 HomeSubscriptionManager 导入

在第 25 行附近找到：
```python
from .core.user import UserManager, MerchantSubscriptionManager
```

替换为：
```python
from .core.user import UserManager, MerchantSubscriptionManager, HomeSubscriptionManager
```

#### B. 在 __init__ 方法中添加家园功能初始化

在第 62 行 `self.merchant_sub_mgr = MerchantSubscriptionManager(data_dir)` 之后添加：

```python
        self.home_sub_mgr = HomeSubscriptionManager(data_dir)
```

在第 68 行 `self.renderer = Renderer(res_path=res_path, render_timeout=render_timeout)` 之后添加：

```python
        self.home_plant_map = self._load_home_plant_map(res_path)
```

在第 90 行 `self._merchant_retry_times = 3` 之后添加：

```python
        self.home_subscription_enabled = self.config.get(
            "home_subscription_enabled", True
        )
        try:
            self.home_subscription_interval_minutes = int(
                self.config.get("home_subscription_interval_minutes", 5) or 5
            )
        except (TypeError, ValueError):
            self.home_subscription_interval_minutes = 5
        self._home_subscription_task = None
```

在第 103 行 merchant subscription task 启动之后添加：

```python
        if self.home_subscription_enabled:
            self._home_subscription_task = asyncio.create_task(
                self._home_subscription_loop()
            )
```

#### C. 在 terminate 方法中添加家园任务清理

在第 184 行 `_merchant_subscription_task` 检查之前添加：

```python
        if hasattr(self, '_home_subscription_task') and self._home_subscription_task and not self._home_subscription_task.done():
            self._home_subscription_task.cancel()
            try:
                await self._home_subscription_task
            except asyncio.CancelledError:
                pass
```

#### D. 添加家园功能辅助方法

在文件中合适位置（建议在 _merchant_check_times 方法之前）添加以下所有辅助方法：

```python
    async def _resolve_home_uid(self, event: AstrMessageEvent, uid: str = "") -> str:
        uid = str(uid or "").strip()
        if uid:
            return uid
        binding = await self.user_mgr.get_primary_binding(event.get_sender_id())
        return str((binding or {}).get("role_id", "") or "")

    def _home_subscription_key(self, session_id: str, uid: str, kind: str) -> str:
        return f"{session_id}:{uid}:{kind}"

    def _normalize_epoch_seconds(self, value: Any) -> int:
        try:
            ts = int(float(value))
        except (TypeError, ValueError):
            return 0
        if ts > 10_000_000_000_000:
            return ts // 1_000_000
        if ts > 10_000_000_000:
            return ts // 1000
        return ts

    def _normalize_duration_seconds(self, value: Any) -> int:
        try:
            seconds = int(float(value))
        except (TypeError, ValueError):
            return 0
        if seconds > 1_000_000_000:
            return seconds // 1_000_000
        if seconds > 1_000_000:
            return seconds // 1000
        return seconds

    def _format_home_remaining(self, target_ts: int, now_ts: int | None = None) -> str:
        if not target_ts:
            return "未开始"
        now_ts = now_ts or int(time.time())
        remain = max(0, int(target_ts) - now_ts)
        if remain <= 0:
            return "已完成"
        hours, remainder = divmod(remain, 3600)
        minutes, _ = divmod(remainder, 60)
        if hours >= 24:
            days, hours = divmod(hours, 24)
            return f"{days}天{hours}小时"
        if hours > 0:
            return f"{hours}小时{minutes}分钟"
        return f"{minutes}分钟"

    def _home_info_payload(self, res: Dict[str, Any] | None) -> Dict[str, Any]:
        payload = res or {}
        if isinstance(payload.get("result"), dict) and isinstance(payload["result"].get("home_info"), dict):
            return payload["result"]["home_info"]
        if isinstance(payload.get("home_info"), dict):
            return payload["home_info"]
        if isinstance(payload.get("data"), dict):
            data = payload["data"]
            if isinstance(data.get("result"), dict) and isinstance(data["result"].get("home_info"), dict):
                return data["result"]["home_info"]
            if isinstance(data.get("home_info"), dict):
                return data["home_info"]
        return payload if isinstance(payload, dict) else {}

    def _home_brief_info(self, home_info: Dict[str, Any]) -> Dict[str, Any]:
        return home_info.get("friend_home_brief_info") or home_info.get("home_brief_info") or home_info or {}

    def _home_cell_info(self, home_info: Dict[str, Any]) -> Dict[str, Any]:
        return home_info.get("friend_cell_home_brief_info") or home_info.get("cell_home_brief_info") or {}

    def _home_pet_icon(self, pet_id: Any, icon_url: str = "") -> str:
        if icon_url:
            return icon_url
        try:
            asset_id = int(str(pet_id))
        except (TypeError, ValueError):
            return ""
        if asset_id <= 0:
            return ""
        if asset_id < 3000:
            asset_id += 3000
        return f"https://game.gtimg.cn/images/rocom/rocodata/jingling/{asset_id}/icon.png"

    def _extract_home_pet(self, raw: Dict[str, Any], index: int, guard: bool = False) -> Dict[str, Any] | None:
        if not isinstance(raw, dict):
            return None
        home_pet = raw.get("home_pet_info") if isinstance(raw.get("home_pet_info"), dict) else raw
        display = raw.get("display_info") if isinstance(raw.get("display_info"), dict) else {}
        pet_id = home_pet.get("pet_cfg_id") or home_pet.get("pet_id") or home_pet.get("pet_base_id") or raw.get("pet_cfg_id") or raw.get("pet_id") or raw.get("id")
        if str(pet_id or "0") in {"", "0"} and not guard:
            return None
        name = home_pet.get("name") or home_pet.get("pet_name") or raw.get("name") or raw.get("pet_name") or f"精灵 {pet_id}"
        feed_info = home_pet.get("feed_info") if isinstance(home_pet.get("feed_info"), dict) else {}
        begin_time = self._normalize_epoch_seconds(feed_info.get("begin_time"))
        time_cost = self._normalize_duration_seconds(feed_info.get("time_cost"))
        rip_time = self._normalize_epoch_seconds(home_pet.get("pet_rip_time") or raw.get("pet_rip_time") or raw.get("rip_time"))
        if not rip_time and begin_time and time_cost:
            rip_time = begin_time + time_cost
        now_ts = int(time.time())
        has_inspiration = bool(rip_time)
        inspire_ready = has_inspiration and now_ts >= rip_time
        status = raw.get("status")
        is_guard = guard or bool(raw.get("is_guard") or raw.get("guard")) or str(status).lower() in {"2", "guard", "守卫"}
        status_text = "守卫中" if is_guard and not has_inspiration else ("灵感已完成" if inspire_ready else ("灵感收集中" if has_inspiration else "未喂食"))
        status_class = "guard" if is_guard and not has_inspiration else ("ready" if inspire_ready else ("progress" if has_inspiration else "idle"))
        return {
            "id": str(pet_id),
            "pos": raw.get("pos") or raw.get("position") or index + 1,
            "name": str(name),
            "level": display.get("level") or raw.get("level") or home_pet.get("level") or "--",
            "iconUrl": self._home_pet_icon(pet_id, raw.get("icon_url") or raw.get("pet_img_url") or raw.get("petIcon") or ""),
            "badge": "守" if is_guard else "",
            "isGuard": is_guard,
            "statusText": status_text,
            "statusClass": status_class,
            "note": self._format_home_remaining(rip_time, now_ts) if has_inspiration else ("家园守卫中" if is_guard else "暂无灵感倒计时"),
            "inspireReady": inspire_ready,
            "readyAt": rip_time,
            "eventId": f"pet:{raw.get('pos') or index + 1}:{pet_id}:{rip_time}",
        }

    def _home_pet_sources(self, home_info: Dict[str, Any]) -> tuple[List[Dict[str, Any]], List[Dict[str, Any]]]:
        cell = self._home_cell_info(home_info)
        indoor_sources = []
        guard_sources = []
        if isinstance(home_info.get("home_pets"), list):
            indoor_sources.extend(home_info.get("home_pets") or [])
        if isinstance(cell.get("home_pets"), list):
            for pet in cell.get("home_pets") or []:
                home_pet = pet.get("home_pet_info") if isinstance(pet, dict) and isinstance(pet.get("home_pet_info"), dict) else {}
                if str(home_pet.get("pet_cfg_id") or "0") == "0" and (home_pet.get("name") or home_pet.get("pet_name")):
                    guard_sources.append(pet)
                else:
                    indoor_sources.append(pet)
        pet_info = cell.get("home_pet_info") if isinstance(cell.get("home_pet_info"), dict) else {}
        if isinstance(pet_info.get("home_pet_list"), list):
            indoor_sources.extend(pet_info.get("home_pet_list") or [])
        for key in ("guard_pets", "home_guard_pets", "guard_pet_list"):
            if isinstance(home_info.get(key), list):
                guard_sources.extend(home_info.get(key) or [])
            if isinstance(cell.get(key), list):
                guard_sources.extend(cell.get(key) or [])
        for key in ("guard_pet", "home_guard_pet", "guard_pet_info", "home_guard_pet_info", "defend_pet", "defend_pet_info", "protect_pet", "protect_pet_info"):
            if isinstance(home_info.get(key), dict):
                guard_sources.append(home_info.get(key))
            if isinstance(cell.get(key), dict):
                guard_sources.append(cell.get(key))
        for key in ("guard_pet_info", "home_guard_pet_info"):
            info = cell.get(key) if isinstance(cell.get(key), dict) else home_info.get(key)
            if isinstance(info, dict):
                for list_key in ("guard_pet_list", "home_guard_pet_list", "pet_list"):
                    if isinstance(info.get(list_key), list):
                        guard_sources.extend(info.get(list_key) or [])
        return indoor_sources, guard_sources

    def _load_home_plant_map(self, res_path: str) -> Dict[str, Any]:
        path = os.path.join(res_path, "render", "home", "data", "home_item_list.json")
        if not os.path.exists(path):
            return {}
        try:
            with open(path, "r", encoding="utf-8", errors="replace") as f:
                data = json.load(f)
            return data if isinstance(data, dict) else {}
        except Exception as e:
            logger.warning(f"[Rocom] 加载家园作物映射失败: {e}")
            return {}

    def _home_plant_icon(self, icon_id: Any) -> str:
        if not icon_id:
            return ""
        icon_text = str(icon_id)
        if icon_text.startswith(("http://", "https://", "data:")):
            return icon_text
        return f"img/home_icon/{icon_text}_2.png"

    def _extract_home_plants(self, home_info: Dict[str, Any]) -> List[Dict[str, Any]]:
        cell = self._home_cell_info(home_info)
        plant_sources = []
        if isinstance(home_info.get("home_plants"), list):
            plant_sources.extend(home_info.get("home_plants") or [])
        plant_info = cell.get("home_plant_info") if isinstance(cell.get("home_plant_info"), dict) else {}
        land_list = plant_info.get("home_plant_land_list") if isinstance(plant_info.get("home_plant_land_list"), list) else []
        for land in land_list:
            if not isinstance(land, dict):
                continue
            for item in land.get("home_plant_list") or []:
                if isinstance(item, dict):
                    copied = dict(item)
                    copied.setdefault("land_index", land.get("land_index"))
                    plant_sources.append(copied)
        now_ts = int(time.time())
        result = []
        for index, raw in enumerate(plant_sources):
            plant_data = raw.get("plant_info") if isinstance(raw.get("plant_info"), dict) else raw
            plant_id = raw.get("plant_seed_id") or raw.get("plant_cfg_id") or raw.get("plant_id") or plant_data.get("id")
            if str(plant_id or "0") in {"", "0"}:
                continue
            mapped_plant = getattr(self, "home_plant_map", {}).get(str(plant_id), {})
            icon_id = (
                plant_data.get("icon_url")
                or plant_data.get("iconUrl")
                or raw.get("icon_url")
                or raw.get("iconUrl")
                or plant_data.get("iconid")
                or raw.get("iconid")
                or raw.get("icon_id")
                or (mapped_plant.get("iconid") if isinstance(mapped_plant, dict) else "")
            )
            rip_time = self._normalize_epoch_seconds(raw.get("plant_rip_time") or raw.get("rip_time") or raw.get("end_time"))
            left_time = int(raw.get("left_time") or 0)
            if not rip_time and left_time > 0:
                rip_time = now_ts + left_time
            ready = bool(rip_time and now_ts >= rip_time) or (raw.get("status") in {2, "ready", "mature"})
            total = int(raw.get("time_cost") or raw.get("total_time") or 0)
            if not total and raw.get("plant_tab_id"):
                try:
                    total = int(raw.get("plant_tab_id")) * 21600
                except (TypeError, ValueError):
                    total = 0
            progress = int(max(0, min(100, ((total - max(0, rip_time - now_ts)) / total) * 100))) if total and rip_time else (100 if ready else 35)
            land_index = raw.get("slot_index") or raw.get("land_index") or index + 1
            harvest_num = raw.get("plant_harvest_num")
            steal_account = raw.get("plant_steal_account")
            can_steal_account = raw.get("plant_can_steal_account")
            result.append({
                "id": str(plant_id),
                "landIndex": land_index,
                "plantName": plant_data.get("name") or raw.get("name") or (mapped_plant.get("name") if isinstance(mapped_plant, dict) else "") or f"种子 {plant_id}",
                "iconUrl": self._home_plant_icon(icon_id),
                "stateType": "ready" if ready else "warning",
                "statusText": "已成熟" if ready else "成长中",
                "leftTimeText": "可收获" if ready else self._format_home_remaining(rip_time, now_ts),
                "progress": progress,
                "ready": ready,
                "readyAt": rip_time,
                "harvestText": f"产量 {harvest_num}" if harvest_num not in (None, "") else "",
                "stealText": f"可偷 {steal_account}/{can_steal_account}" if steal_account not in (None, "") and can_steal_account not in (None, "") else "",
                "eventId": f"plant:{raw.get('slot_index') or raw.get('land_index') or index}:{plant_id}:{rip_time}",
            })
        return result

    def _build_home_render_data(self, res: Dict[str, Any] | None, uid: str) -> Dict[str, Any]:
        home_info = self._home_info_payload(res)
        brief = self._home_brief_info(home_info)
        indoor_sources, guard_sources = self._home_pet_sources(home_info)
        indoor_pets = []
        guard_pets = []
        for index, raw in enumerate(indoor_sources):
            item = self._extract_home_pet(raw, index)
            if not item:
                continue
            if item["isGuard"]:
                guard_pets.append(item)
            else:
                indoor_pets.append(item)
        for index, raw in enumerate(guard_sources):
            item = self._extract_home_pet(raw, index, guard=True)
            if item:
                guard_pets.append(item)
        garden_plots = self._extract_home_plants(home_info)
        home_name = brief.get("home_name") or brief.get("name") or f"{uid} 的小屋"
        meta = (res or {}).get("meta") or {}
        created_at = self._normalize_epoch_seconds(meta.get("created_at"))
        updated_at = datetime.fromtimestamp(created_at, tz=self._cn_tz()).strftime("%Y-%m-%d %H:%M:%S") if created_at else datetime.now(self._cn_tz()).strftime("%Y-%m-%d %H:%M:%S")
        return {
            "title": "洛克家园",
            "subtitle": "Home Information",
            "homeName": home_name,
            "uid": uid,
            "summaryCards": [
                {"label": "房间等级", "value": brief.get("room_level", "--")},
                {"label": "家园等级", "value": brief.get("home_level", "--")},
                {"label": "家园经验", "value": brief.get("home_experience", "--")},
                {"label": "舒适度", "value": brief.get("home_comfort_level", "--")},
            ],
            "gardenPlots": garden_plots,
            "guardPets": guard_pets,
            "indoorPets": indoor_pets,
            "gardenCount": len(garden_plots),
            "guardCount": len(guard_pets),
            "indoorCount": len(indoor_pets),
            "guardEmptyText": "后端当前返回中没有守卫精灵字段",
            "updatedAt": updated_at,
        }

    async def _home_subscription_loop(self):
        logger.info("[Rocom] 家园订阅循环任务已启动")
        interval = max(1, int(self.home_subscription_interval_minutes or 5)) * 60
        while True:
            try:
                await asyncio.sleep(interval)
                await self._check_home_subscriptions()
            except asyncio.CancelledError:
                raise
            except Exception as e:
                logger.error(f"[Rocom] 家园订阅循环异常: {e}")
                await asyncio.sleep(60)

    def _home_subscription_state(
        self, data: Dict[str, Any], kind: str
    ) -> tuple[List[Dict[str, Any]], List[Dict[str, Any]], str, List[str]]:
        if kind == "garden":
            items = list(data.get("gardenPlots") or [])
            ready_items = [item for item in items if item.get("ready")]
            unit = "成熟"
            names = [f"田地{item.get('landIndex')} {item.get('plantName')}" for item in ready_items]
            return items, ready_items, unit, names

        items = [
            item
            for item in list(data.get("indoorPets") or []) + list(data.get("guardPets") or [])
            if item.get("readyAt")
        ]
        ready_items = [item for item in items if item.get("inspireReady")]
        unit = "灵感完成"
        names = [item.get("name", "未知精灵") for item in ready_items]
        return items, ready_items, unit, names

    def _home_subscription_level_message(
        self,
        uid: str,
        kind: str,
        level: str,
        total_count: int,
        ready_items: List[Dict[str, Any]],
        names: List[str],
    ) -> str:
        kind_text = "菜园作物" if kind == "garden" else "精灵灵感"
        action_text = "成熟" if kind == "garden" else "完成"
        level_text = "首个" if level == "first" else "全部"
        title = f"家园{kind_text}{level_text}{action_text}提醒"
        lines = [
            f"{title}：{uid}",
            f"进度：{len(ready_items)}/{total_count}",
        ]
        if names:
            lines.append("已完成：" + "、".join(names[:8]))
        return "\n".join(lines)

    async def _check_home_subscriptions(self):
        all_subs = await self.home_sub_mgr.get_all_subscriptions()
        if not all_subs:
            return
        data_cache: Dict[str, Dict[str, Any] | None] = {}
        for key, sub in all_subs.items():
            uid = str(sub.get("uid", "") or "")
            kind = str(sub.get("kind", "") or "")
            if not uid or kind not in {"garden", "inspiration"}:
                continue
            if uid not in data_cache:
                data_cache[uid] = await self.client.ingame_home_info(uid)
                await asyncio.sleep(1)
            res = data_cache.get(uid)
            if not res:
                continue
            data = self._build_home_render_data(res, uid)
            total_items, ready_items, _unit, names = self._home_subscription_state(data, kind)
            total_count = len(total_items)
            ready_count = len(ready_items)
            if total_count <= 0:
                continue

            notify_state = sub.get("notify_state") if isinstance(sub.get("notify_state"), dict) else {}
            changed = False
            push_levels = []

            if ready_count <= 0:
                if notify_state.get("first") or notify_state.get("all"):
                    notify_state["first"] = False
                    notify_state["all"] = False
                    changed = True
            else:
                if not notify_state.get("first"):
                    push_levels.append("first")
                if ready_count >= total_count and not notify_state.get("all"):
                    push_levels.append("all")
                elif ready_count < total_count and notify_state.get("all"):
                    notify_state["all"] = False
                    changed = True

            if not push_levels:
                if changed:
                    sub["notify_state"] = notify_state
                    await self.home_sub_mgr.upsert_subscription(key, sub)
                continue

            messages = [
                self._home_subscription_level_message(uid, kind, level, total_count, ready_items, names)
                for level in push_levels
            ]
            try:
                await self.context.send_message(sub["umo"], MessageChain().message("\n\n".join(messages)))
            except Exception as e:
                logger.warning(f"[Rocom] 家园订阅推送失败: {e}")
                continue
            for level in push_levels:
                notify_state[level] = True
            sub["notify_state"] = notify_state
            sub["last_push_time"] = int(time.time())
            await self.home_sub_mgr.upsert_subscription(key, sub)
            await asyncio.sleep(2)
```

#### E. 添加家园功能命令

在文件中合适位置（建议在 /洛克商店 命令之前）添加以下命令：

```python
    @filter.command("洛克家园")
    async def rocom_home(self, event: AstrMessageEvent, uid: str = ""):
        """通过 UID 查询洛克家园菜园、守卫精灵与室内精灵"""
        uid = await self._resolve_home_uid(event, uid)
        if not uid:
            yield event.plain_result("请提供玩家 UID，或先完成绑定后使用 /洛克家园。")
            return
        res = await self.client.ingame_home_info(uid)
        if not res:
            yield event.plain_result(f"家园查询失败：{self.client.get_last_error()}")
            return
        data = self._build_home_render_data(res, uid)
        img_url = await self.renderer.render_html(
            "render/home/index.html",
            data,
            {
                "device_scale_factor": 3,
                "viewport_width": 1500,
                "viewport_height": 1200,
            },
        )
        if img_url:
            yield event.image_result(img_url)
        else:
            yield event.plain_result(self._format_json_payload(res))

    @filter.command("订阅家园菜园")
    async def subscribe_home_garden(self, event: AstrMessageEvent, uid: str = ""):
        """订阅家园菜园成熟提醒"""
        if not event.is_private_chat() and not await self._is_group_admin(event):
            yield event.plain_result("仅当前群管理员可以配置家园菜园订阅。")
            return
        uid = await self._resolve_home_uid(event, uid)
        if not uid:
            yield event.plain_result("请提供玩家 UID，或先完成绑定后再订阅家园菜园。")
            return
        key = self._home_subscription_key(event.unified_msg_origin, uid, "garden")
        await self.home_sub_mgr.upsert_subscription(
            key,
            {
                "key": key,
                "kind": "garden",
                "uid": uid,
                "umo": event.unified_msg_origin,
                "updated_by": str(event.get_sender_id()),
                "sent_event_ids": [],
                "notify_state": {"first": False, "all": False},
                "updated_at": int(time.time()),
            },
        )
        yield event.plain_result(f"已订阅 UID {uid} 的家园菜园提醒：首个成熟和全部成熟时各推送一次。")

    @filter.command("订阅家园灵感")
    async def subscribe_home_inspiration(self, event: AstrMessageEvent, uid: str = ""):
        """订阅家园精灵灵感完成提醒"""
        if not event.is_private_chat() and not await self._is_group_admin(event):
            yield event.plain_result("仅当前群管理员可以配置家园灵感订阅。")
            return
        uid = await self._resolve_home_uid(event, uid)
        if not uid:
            yield event.plain_result("请提供玩家 UID，或先完成绑定后再订阅家园灵感。")
            return
        key = self._home_subscription_key(event.unified_msg_origin, uid, "inspiration")
        await self.home_sub_mgr.upsert_subscription(
            key,
            {
                "key": key,
                "kind": "inspiration",
                "uid": uid,
                "umo": event.unified_msg_origin,
                "updated_by": str(event.get_sender_id()),
                "sent_event_ids": [],
                "notify_state": {"first": False, "all": False},
                "updated_at": int(time.time()),
            },
        )
        yield event.plain_result(f"已订阅 UID {uid} 的家园精灵灵感提醒：首个完成和全部完成时各推送一次。")

    @filter.command("取消订阅家园")
    async def unsubscribe_home(self, event: AstrMessageEvent, kind: str = "全部", uid: str = ""):
        """取消家园菜园或灵感订阅"""
        if not event.is_private_chat() and not await self._is_group_admin(event):
            yield event.plain_result("仅当前群管理员可以取消家园订阅。")
            return
        kind_map = {
            "菜园": "garden",
            "灵感": "inspiration",
            "全部": "",
            "all": "",
            "garden": "garden",
            "inspiration": "inspiration",
        }
        selected_kind = kind_map.get(str(kind or "全部").strip(), "")
        deleted = await self.home_sub_mgr.delete_matching(
            event.unified_msg_origin,
            kind=selected_kind,
            uid=str(uid or "").strip(),
        )
        if deleted:
            yield event.plain_result(f"已取消 {deleted} 条家园订阅。")
        else:
            yield event.plain_result("当前会话没有匹配的家园订阅。")
```

### 2. _conf_schema.json

在配置 schema 中添加家园订阅相关配置项（在 merchant_subscription 配置之后）：

```json
    "home_subscription_enabled": {
      "type": "boolean",
      "default": true,
      "description": "是否启用家园订阅功能"
    },
    "home_subscription_interval_minutes": {
      "type": "integer",
      "default": 5,
      "description": "家园订阅检查间隔（分钟）"
    }
```

## 已完成的工作

以下文件已从上游合并，无需手动操作：
- ✅ core/client.py - 已添加 ingame_home_info() 方法
- ✅ core/user.py - 已添加 HomeSubscriptionManager 类
- ✅ render/home/ - 完整的家园渲染模板目录
- ✅ metadata.yaml - 版本信息更新
- ✅ README.md - 文档更新

## 测试建议

应用补丁后，建议测试以下功能：
1. `/洛克家园 [UID]` - 查询家园信息
2. `/订阅家园菜园 [UID]` - 订阅菜园提醒
3. `/订阅家园灵感 [UID]` - 订阅灵感提醒
4. `/取消订阅家园 [类型] [UID]` - 取消订阅
5. 检查插件启动日志，确认家园订阅任务正常启动

## 注意事项

1. 本补丁保留了 pr0.4 分支的所有现有功能（包括 Wiki 功能）
2. 家园功能依赖 render/home/ 目录下的模板和资源文件
3. 确保数据库文件 rocom_home_subscriptions.json 有写入权限
4. 家园订阅功能默认启用，可通过配置项关闭
