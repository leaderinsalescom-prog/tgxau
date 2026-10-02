# Architecture rules

- broker_accounts is the single source of truth for MT5 accounts; settings.mt5_* is a trigger-maintained mirror — avoids two conflicting credential stores.
- VPS scripts read all settings live from the cloud (settings row, polled every 2s); never bake config into the ZIP — changes apply without re-downloading.
- Layering state lives in public.signal_layers, unique per (signal_id, account_key, layer_no), claimed via compare-and-set status updates — prevents duplicate orders across restarts/re-deliveries.
- Layering maths lives in public/scripts/layering.py (pure, no MT5) so it can be unit-tested off Windows.
- handle_open builds one plan (levels → execute_plan) for every mode; every order carries the single signal TP + SL — Layering/Marginal Entry/Execution Mode are independent switches on that one path, so settings never override each other.
- Bridges poll only their own pending_actions (payload->>broker_account_id) and push non-critical cloud writes to a background thread — an offline account never blocks others and DB latency never sits before an MT5 order.
