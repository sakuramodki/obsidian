# SLO駆動型インシデント検知システム (SLO-Driven Incident Detection System)

## モデル概要
Service Level Objective (SLO) を基軸とした顧客影響度重視のインシデント検知・対応システム。技術指標ではなく顧客体験指標に基づくアラート設計により、意味のある通知のみを生成する。

## モデル適用条件
- **顧客体験重視**: 顧客影響度が最重要指標
- **SLO設定可能**: 明確な顧客期待値・サービス品質基準
- **メトリクス収集基盤**: 顧客体験指標の継続的収集が可能
- **誤報削減要求**: アラート疲労・重要通知見逃しリスクの回避

## システム構成要素

### SLI (Service Level Indicator) 設計パターン

#### パターン1: 可用性SLI
```python
class AvailabilitySLI:
    """可用性に基づくSLI実装"""
    
    def __init__(self, service_name: str):
        self.service_name = service_name
        
    def calculate_sli(self, time_window: timedelta) -> float:
        """指定期間の可用性SLI計算"""
        success_count = self._get_success_requests(time_window)
        total_count = self._get_total_requests(time_window)
        
        if total_count == 0:
            return 1.0  # リクエストがない場合は100%とみなす
            
        return success_count / total_count
    
    def _get_success_requests(self, time_window: timedelta) -> int:
        """成功リクエスト数取得"""
        query = f"""
        SELECT COUNT(*)
        FROM request_logs 
        WHERE service = '{self.service_name}'
          AND status_code < 500
          AND timestamp >= NOW() - INTERVAL '{time_window.total_seconds()}' SECOND
        """
        return self._execute_query(query)
    
    def _get_total_requests(self, time_window: timedelta) -> int:
        """総リクエスト数取得"""  
        query = f"""
        SELECT COUNT(*)
        FROM request_logs
        WHERE service = '{self.service_name}'
          AND timestamp >= NOW() - INTERVAL '{time_window.total_seconds()}' SECOND
        """
        return self._execute_query(query)

# 使用例
availability_sli = AvailabilitySLI("payment-service")
current_availability = availability_sli.calculate_sli(timedelta(minutes=5))
```

#### パターン2: レイテンシSLI
```python
class LatencySLI:
    """レイテンシに基づくSLI実装"""
    
    def __init__(self, service_name: str, percentile: float = 95.0):
        self.service_name = service_name
        self.percentile = percentile
        
    def calculate_sli(self, time_window: timedelta) -> float:
        """指定期間のレイテンシSLI計算"""
        latencies = self._get_response_times(time_window)
        
        if not latencies:
            return 0.0
            
        percentile_latency = self._calculate_percentile(latencies, self.percentile)
        return percentile_latency
    
    def _get_response_times(self, time_window: timedelta) -> List[float]:
        """レスポンス時間一覧取得"""
        query = f"""
        SELECT response_time_ms
        FROM request_logs
        WHERE service = '{self.service_name}'
          AND status_code < 500
          AND timestamp >= NOW() - INTERVAL '{time_window.total_seconds()}' SECOND
        ORDER BY response_time_ms
        """
        return self._execute_query(query)
    
    def _calculate_percentile(self, values: List[float], percentile: float) -> float:
        """パーセンタイル値計算"""
        if not values:
            return 0.0
            
        sorted_values = sorted(values)
        index = int((percentile / 100.0) * len(sorted_values))
        index = min(index, len(sorted_values) - 1)
        
        return sorted_values[index]

# 使用例
latency_sli = LatencySLI("api-gateway", percentile=95.0)
p95_latency = latency_sli.calculate_sli(timedelta(minutes=5))
```

#### パターン3: Critical User Journey SLI
```python
class UserJourneySLI:
    """ユーザージャーニーに基づくSLI実装"""
    
    def __init__(self, journey_name: str):
        self.journey_name = journey_name
        self.journey_steps = self._load_journey_definition()
        
    def calculate_sli(self, time_window: timedelta) -> Dict[str, float]:
        """ユーザージャーニー全体のSLI計算"""
        journey_results = self._get_journey_results(time_window)
        
        return {
            "success_rate": self._calculate_success_rate(journey_results),
            "completion_time": self._calculate_completion_time(journey_results),
            "step_success_rates": self._calculate_step_success_rates(journey_results)
        }
    
    def _load_journey_definition(self) -> List[Dict]:
        """ジャーニー定義の読み込み"""
        # 設定ファイルまたはデータベースからジャーニー定義を読み込み
        return [
            {"step": "login", "max_duration_ms": 2000},
            {"step": "product_search", "max_duration_ms": 3000},
            {"step": "add_to_cart", "max_duration_ms": 1000},
            {"step": "checkout", "max_duration_ms": 5000},
            {"step": "payment", "max_duration_ms": 10000}
        ]
    
    def _get_journey_results(self, time_window: timedelta) -> List[Dict]:
        """ジャーニー実行結果取得"""
        query = f"""
        SELECT journey_id, step_name, success, duration_ms, timestamp
        FROM user_journey_logs
        WHERE journey_name = '{self.journey_name}'
          AND timestamp >= NOW() - INTERVAL '{time_window.total_seconds()}' SECOND
        ORDER BY journey_id, step_order
        """
        return self._execute_query(query)
    
    def _calculate_success_rate(self, journey_results: List[Dict]) -> float:
        """ジャーニー成功率計算"""
        journey_groups = self._group_by_journey_id(journey_results)
        
        successful_journeys = 0
        total_journeys = len(journey_groups)
        
        for journey_id, steps in journey_groups.items():
            if all(step['success'] for step in steps):
                successful_journeys += 1
                
        return successful_journeys / total_journeys if total_journeys > 0 else 0.0

# 使用例
user_journey_sli = UserJourneySLI("checkout_flow")
journey_metrics = user_journey_sli.calculate_sli(timedelta(hours=1))
```

### SLO定義・管理システム

```python
class SLOManager:
    """SLO定義・管理システム"""
    
    def __init__(self, config_path: str):
        self.slos = self._load_slo_config(config_path)
        self.error_budget_calculator = ErrorBudgetCalculator()
        
    def _load_slo_config(self, config_path: str) -> Dict:
        """SLO設定読み込み"""
        # YAML形式でのSLO定義例
        slo_config = {
            "payment_service": {
                "availability": {
                    "target": 99.9,  # 99.9%
                    "window": "30d",
                    "sli_type": "success_rate"
                },
                "latency": {
                    "target": 200,   # 200ms
                    "percentile": 95,
                    "window": "30d",
                    "sli_type": "latency"
                }
            },
            "user_checkout": {
                "journey_success": {
                    "target": 99.5,  # 99.5%
                    "window": "7d", 
                    "sli_type": "user_journey"
                }
            }
        }
        return slo_config
    
    def evaluate_slo_compliance(self, service: str, slo_type: str) -> Dict:
        """SLO遵守状況評価"""
        slo_config = self.slos[service][slo_type]
        
        # 現在のSLI値取得
        current_sli = self._get_current_sli(service, slo_type, slo_config)
        
        # エラーバジェット計算
        error_budget_status = self.error_budget_calculator.calculate_status(
            slo_config, current_sli
        )
        
        return {
            "service": service,
            "slo_type": slo_type,
            "target": slo_config["target"],
            "current_value": current_sli,
            "compliance": current_sli >= slo_config["target"],
            "error_budget_remaining": error_budget_status["remaining_percentage"],
            "burn_rate": error_budget_status["burn_rate"]
        }

class ErrorBudgetCalculator:
    """エラーバジェット計算システム"""
    
    def calculate_status(self, slo_config: Dict, current_sli: float) -> Dict:
        """エラーバジェット状況計算"""
        target = slo_config["target"]
        window_days = self._parse_window(slo_config["window"])
        
        # エラーバジェット計算
        error_budget = 100 - target  # 例: 99.9% SLO → 0.1% エラーバジェット
        current_error_rate = 100 - current_sli
        
        # 消費率計算
        consumption_rate = current_error_rate / error_budget if error_budget > 0 else 0
        
        # バーンレート計算（1時間あたりの消費率）
        burn_rate = consumption_rate * (1 / (window_days * 24))
        
        return {
            "total_budget": error_budget,
            "consumed": current_error_rate,
            "remaining_percentage": max(0, (error_budget - current_error_rate) / error_budget * 100),
            "burn_rate": burn_rate,
            "estimated_depletion": self._estimate_depletion_time(burn_rate, error_budget - current_error_rate)
        }
    
    def _estimate_depletion_time(self, burn_rate: float, remaining_budget: float) -> Optional[timedelta]:
        """エラーバジェット枯渇予想時間"""
        if burn_rate <= 0 or remaining_budget <= 0:
            return None
            
        hours_to_depletion = remaining_budget / burn_rate
        return timedelta(hours=hours_to_depletion)
```

### アラート生成・管理システム

```python
class SLOAlertManager:
    """SLOベースアラート管理システム"""
    
    def __init__(self, slo_manager: SLOManager):
        self.slo_manager = slo_manager
        self.alert_rules = self._load_alert_rules()
        self.notification_manager = NotificationManager()
        
    def _load_alert_rules(self) -> Dict:
        """アラートルール定義"""
        return {
            "fast_burn": {
                "condition": "burn_rate > 14.4 AND duration >= 2m",
                "severity": "critical",
                "description": "2%のエラーバジェットを1時間で消費する燃焼率",
                "notification_channels": ["pager", "slack"]
            },
            "slow_burn": {
                "condition": "burn_rate > 6 AND duration >= 15m", 
                "severity": "warning",
                "description": "5%のエラーバジェットを6時間で消費する燃焼率",
                "notification_channels": ["slack", "email"]
            },
            "budget_exhaustion": {
                "condition": "error_budget_remaining < 10%",
                "severity": "major",
                "description": "エラーバジェット残量10%未満",
                "notification_channels": ["slack", "email"]
            }
        }
    
    async def evaluate_alerts(self) -> List[Dict]:
        """アラート評価・生成"""
        active_alerts = []
        
        for service, slo_types in self.slo_manager.slos.items():
            for slo_type in slo_types.keys():
                slo_status = self.slo_manager.evaluate_slo_compliance(service, slo_type)
                
                # 各アラートルールを評価
                for rule_name, rule_config in self.alert_rules.items():
                    if self._evaluate_alert_condition(rule_config["condition"], slo_status):
                        alert = self._create_alert(service, slo_type, rule_name, rule_config, slo_status)
                        active_alerts.append(alert)
                        
                        # 通知送信
                        await self._send_alert_notifications(alert)
        
        return active_alerts
    
    def _evaluate_alert_condition(self, condition: str, slo_status: Dict) -> bool:
        """アラート条件評価"""
        # 簡略化された条件評価（実際にはより詳細な実装が必要）
        burn_rate = slo_status.get("burn_rate", 0)
        error_budget_remaining = slo_status.get("error_budget_remaining", 100)
        
        if "burn_rate > 14.4" in condition:
            return burn_rate > 14.4
        elif "burn_rate > 6" in condition:
            return burn_rate > 6
        elif "error_budget_remaining < 10%" in condition:
            return error_budget_remaining < 10
            
        return False
    
    def _create_alert(self, service: str, slo_type: str, rule_name: str, 
                     rule_config: Dict, slo_status: Dict) -> Dict:
        """アラート作成"""
        return {
            "id": self._generate_alert_id(),
            "service": service,
            "slo_type": slo_type,
            "rule": rule_name,
            "severity": rule_config["severity"],
            "description": rule_config["description"],
            "current_sli": slo_status["current_value"],
            "target_slo": slo_status["target"],
            "error_budget_remaining": slo_status["error_budget_remaining"],
            "burn_rate": slo_status["burn_rate"],
            "timestamp": datetime.utcnow(),
            "notification_channels": rule_config["notification_channels"]
        }
    
    async def _send_alert_notifications(self, alert: Dict) -> None:
        """アラート通知送信"""
        for channel in alert["notification_channels"]:
            await self.notification_manager.send_notification(channel, alert)

class NotificationManager:
    """通知管理システム"""
    
    async def send_notification(self, channel: str, alert: Dict) -> None:
        """チャネル別通知送信"""
        if channel == "slack":
            await self._send_slack_notification(alert)
        elif channel == "pager":
            await self._send_pager_notification(alert)
        elif channel == "email":
            await self._send_email_notification(alert)
    
    async def _send_slack_notification(self, alert: Dict) -> None:
        """Slack通知送信"""
        message = self._format_slack_message(alert)
        # Slack API呼び出し実装
        pass
    
    def _format_slack_message(self, alert: Dict) -> Dict:
        """Slack通知フォーマット"""
        severity_emoji = {
            "critical": "🚨",
            "major": "⚠️",
            "warning": "⚡"
        }
        
        return {
            "blocks": [
                {
                    "type": "header",
                    "text": {
                        "type": "plain_text",
                        "text": f"{severity_emoji[alert['severity']]} SLO Alert: {alert['service']}"
                    }
                },
                {
                    "type": "section",
                    "fields": [
                        {
                            "type": "mrkdwn",
                            "text": f"*Service:* {alert['service']}"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*SLO Type:* {alert['slo_type']}"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Current SLI:* {alert['current_sli']:.3f}%"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Target SLO:* {alert['target_slo']:.3f}%"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Error Budget:* {alert['error_budget_remaining']:.1f}% remaining"
                        },
                        {
                            "type": "mrkdwn", 
                            "text": f"*Burn Rate:* {alert['burn_rate']:.2f}x"
                        }
                    ]
                },
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": f"*Description:* {alert['description']}"
                    }
                }
            ]
        }
```

### システム統合・運用

```python
class SLODrivenIncidentDetection:
    """SLO駆動型インシデント検知システム統合"""
    
    def __init__(self, config_path: str):
        self.slo_manager = SLOManager(config_path)
        self.alert_manager = SLOAlertManager(self.slo_manager)
        self.incident_manager = IncidentManager()
        
    async def run_detection_cycle(self) -> None:
        """検知サイクル実行"""
        while True:
            try:
                # SLO評価・アラート生成
                active_alerts = await self.alert_manager.evaluate_alerts()
                
                # クリティカルアラートのインシデント化
                for alert in active_alerts:
                    if alert["severity"] == "critical":
                        await self._create_incident_from_alert(alert)
                
                # 次回実行まで待機
                await asyncio.sleep(60)  # 1分間隔
                
            except Exception as e:
                logger.error(f"Detection cycle error: {e}")
                await asyncio.sleep(60)
    
    async def _create_incident_from_alert(self, alert: Dict) -> None:
        """アラートからインシデント作成"""
        incident = {
            "title": f"SLO Violation: {alert['service']} {alert['slo_type']}",
            "description": self._generate_incident_description(alert),
            "severity": self._map_severity(alert["severity"]),
            "service": alert["service"],
            "alert_id": alert["id"],
            "created_at": datetime.utcnow()
        }
        
        await self.incident_manager.create_incident(incident)
    
    def _generate_incident_description(self, alert: Dict) -> str:
        """インシデント説明文生成"""
        return f"""
SLO violation detected for {alert['service']} ({alert['slo_type']})

Current SLI: {alert['current_sli']:.3f}%
Target SLO: {alert['target_slo']:.3f}%
Error Budget Remaining: {alert['error_budget_remaining']:.1f}%
Burn Rate: {alert['burn_rate']:.2f}x

Alert Rule: {alert['rule']}
Description: {alert['description']}

Please investigate and take appropriate action to restore service level.
        """.strip()

# 使用例・設定例
if __name__ == "__main__":
    # システム初期化
    detection_system = SLODrivenIncidentDetection("slo_config.yaml")
    
    # 検知ループ開始
    asyncio.run(detection_system.run_detection_cycle())
```

## 設定ファイル例

```yaml
# slo_config.yaml
services:
  payment_service:
    availability:
      target: 99.9
      window: 30d
      sli_type: success_rate
      alert_rules:
        - fast_burn
        - slow_burn
        - budget_exhaustion
    
    latency:
      target: 200
      percentile: 95
      window: 30d
      sli_type: latency
      alert_rules:
        - fast_burn
        - slow_burn

  user_api:
    availability:
      target: 99.95
      window: 30d
      sli_type: success_rate
    
    user_journey:
      target: 99.5
      window: 7d
      sli_type: user_journey
      journey_name: "user_registration"

alert_channels:
  slack:
    webhook_url: "https://hooks.slack.com/services/..."
    channel: "#incidents"
  
  pager:
    service_key: "your-pagerduty-service-key"
  
  email:
    smtp_server: "smtp.company.com"
    recipients: ["sre-team@company.com"]
```

## 実装ベストプラクティス

### 1. SLI設計原則
- **顧客視点**: 顧客が実際に体験する指標を選択
- **測定可能性**: 継続的に自動測定可能な指標
- **関連性**: ビジネス価値と直接関連する指標

### 2. アラート精度向上
- **バーンレートベース**: エラーバジェット消費速度による段階的アラート
- **複数時間窓**: 短期・長期の複数時間窓での評価
- **コンテキスト考慮**: 時間帯・曜日・季節性の考慮

### 3. 運用効率化
- **自動化**: 定型的な評価・通知の完全自動化
- **可視化**: SLO達成状況の直感的なダッシュボード
- **統合**: 既存監視システムとの適切な統合

このモデルにより、技術指標に惑わされない、真に顧客価値を重視したインシデント検知システムを構築できる。

## 関連事例

### 実装事例
- [メルペイにおける6年間のインシデント対応・管理で直面した課題と改善 | メルカリエンジニアリング](https://engineering.mercari.com/blog/entry/20250617-56adf5904e/)
  - SLOベースアラートの本格導入事例
  - CPU/メモリ使用率から顧客影響度指標への転換
  - 「ページャーが鳴る = 顧客影響発生」状態の実現

### 技術詳細事例
- [E2E Testを用いたマイクロサービスアーキテクチャでのUser Journey SLOの継続的最新化](https://engineering.mercari.com/blog/entry/20241204-keeping-user-journey-slos-up-to-date-with-e2e-testing-in-a-microservices-architecture/)
  - Critical User Journey SLIの実装詳細
  - E2Eテスト連携したSLO自動更新システム

### AI活用事例
- [LLM x SRE: メルカリの次世代インシデント対応](https://engineering.mercari.com/blog/entry/20250206-llm-sre-incident-handling-buddy/)
  - AI支援でのアラート分析・削減提案
  - 機械学習を活用した異常検知精度向上

### 関連モデル  
- [信頼性主導型競争優位モデル](../01_Context/SRE/信頼性主導型競争優位モデル.md) - SLO達成が競争優位につながる戦略モデル
- [分散責任型インシデント管理アーキテクチャ](../02_Container/SRE/分散責任型インシデント管理アーキテクチャ.md) - SLO監視を支える組織アーキテクチャ
- [学習駆動型インシデント改善モデル](../03_Component/SRE/学習駆動型インシデント改善モデル.md) - SLOデータを活用した継続改善