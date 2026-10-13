# Scenario 5: Emergency Production Rollback & Auto-Healing Pipeline via GitLab CI/CD & ArgoCD GitOps

> **Production Context:** သောကြာနေ့ညနေ Production Release တင်ပြီး မိနစ်ပိုင်းအကြာတွင် Memory Leak ဖြစ်ပွားပြီး Database Connection Pool ပြည့်သွားကာ HTTP 500 Error Rate သည် ၂၅% အထိ ခုန်တက်သွားပါသည်။ Site Reliability Engineering (SRE) စံနှုန်းအရ Production Down-time ကို ၁၅ စက္ကန့်အတွင်း ပြန်လည်ကုစားရန် **GitLab CI/CD နှင့် ArgoCD GitOps Auto-Healing Rollback Pipeline** ကို အရေးပေါ်လည်ပတ်ခဲ့ပုံ။

---

## 1. The Anti-Pattern: Why `kubectl edit` / `kubectl rollout undo` Fails in GitOps

DevOps အင်ဂျင်နီယာအများစု အရေးပေါ်အခြေအနေတွင် ပြုလုပ်လေ့ရှိသည့် အမှားမှာ Production Cluster ထဲသို့ `kubectl edit deployment` သို့မဟုတ် `kubectl rollout undo` ရိုက်ထည့်ခြင်း ဖြစ်ပါသည်။

```mermaid
flowchart TD
    subgraph AntiPattern [Manual kubectl Rollout: Drift Conflict]
        Eng[Engineer runs: kubectl rollout undo] --> K8s[K8s Cluster State Changes]
        Argo[ArgoCD GitOps Controller] -->|Detects OutOfSync with Git!| Argo
        Argo -->|Auto-Heal / Self-Healing kicks in| K8s
        Argo -- Reverts back to Bad Image! --> K8s
        note1[ArgoCD သည် Git ကို Single Source of Truth အဖြစ် မှတ်ယူထားသဖြင့်<br/>Manual အပြောင်းအလဲများကို Bad Commit အတိုင်း ပြန်လည် Overwrite လုပ်ပစ်သည်!]
    end
```

### GitOps Rule of Law:
> **"Git Repository is the Single Source of Truth."**
> မည်သည့် Rollback ဖြစ်စေ Cluster ပေါ်တွင် လက်ဖြင့်ပြင်ဆင်ခြင်း မပြုရပါ။ **Git Repository ရှိ Manifest (Image Tag) ကို ပြောင်းလဲပေးမှသာလျှင်** ArgoCD သည် လုံခြုံစွာဖြင့် Zero-Drift Rollback ကို ပြုလုပ်ပေးနိုင်မည် ဖြစ်ပါသည်။

---

## 2. Solution: Automated Instant GitOps Rollback Architecture

GitLab CI/CD တွင် တစ်ချက်နှိပ်ရုံဖြင့် ယခင် Stable Image Tag သို့ Git commit အသစ် auto-generate ပြုလုပ်ပေးပြီး ArgoCD API Webhook သို့ Sync Signal ပေးပို့သည့် **Emergency Rollback Pipeline** ကို တည်ဆောက်ထားပါသည်။

```mermaid
sequenceDiagram
    autonumber
    actor SRE as SRE / On-Call Engineer
    participant GL as GitLab CI Rollback Job
    participant GitOps as GitOps Repo (k8s-manifests)
    participant Argo as ArgoCD Controller
    participant K8s as Production K8s Cluster

    SRE->>GL: Trigger 'emergency:rollback' Job (Select Target Tag: v2.4.1)
    GL->>GitOps: Clone & Patch deployment.yaml (image: v2.4.1)
    GL->>GitOps: Git Commit & Push: "chore(rollback): Emergency revert to v2.4.1"
    GL->>Argo: POST /api/v1/applications/prod-app/sync (Instant Webhook)
    Argo->>GitOps: Fetch Latest Git Commit
    Argo->>K8s: Execute Rolling Update to v2.4.1 (Under 15 Seconds!)
    K8s-->>SRE: Health Checks 200 OK. Error Rate Drops to 0%!
```

---

## 3. Emergency Rollback `.gitlab-ci.yml` Pipeline

အောက်ပါ Pipeline သည် App Repo မှနေ၍ GitOps Configuration Repo ဆီသို့ Git Automation ဖြင့် Instant Rollback ပြုလုပ်ပေးသော စနစ်ဖြစ်ပါသည်။

```yaml
# ==============================================================================
# Production Emergency Rollback Pipeline (GitLab CI -> ArgoCD)
# ==============================================================================

stages:
  - rollback-trigger
  - health-verification

variables:
  GITOPS_REPO_URL: "https://gitlab-ci-token:${GITOPS_ACCESS_TOKEN}@gitlab.company.com/platform/gitops-manifests.git"
  APP_NAME: "order-service"
  ARGOCD_SERVER: "argocd.company.internal"

# ------------------------------------------------------------------------------
# Emergency Rollback Job (Manual One-Click Execution)
# ------------------------------------------------------------------------------
emergency:rollback:
  stage: rollback-trigger
  image: alpine/git:2.43.0
  before_script:
    - apk add --no-cache curl jq yq
    - git config --global user.email "ci-automation@company.com"
    - git config --global user.name "GitLab CI Auto-Rollback"
  script:
    - echo "🚨 INITIATING EMERGENCY GITOPS ROLLBACK..."
    - |
      if [ -z "$ROLLBACK_TARGET_TAG" ]; then
        echo "❌ ERROR: ROLLBACK_TARGET_TAG variable must be provided (e.g., v1.9.8)!"
        exit 1
      fi
    # 1. Clone GitOps Repository
    - git clone --depth 1 ${GITOPS_REPO_URL} gitops-repo
    - cd gitops-repo/apps/${APP_NAME}/overlays/production
    
    # 2. Update Image Tag using YQ (Zero compilation time)
    - echo "Updating target image tag to: $ROLLBACK_TARGET_TAG"
    - yq eval ".spec.template.spec.containers[0].image = \"$CI_REGISTRY_IMAGE:$ROLLBACK_TARGET_TAG\"" -i deployment.yaml
    
    # 3. Commit and Push back to GitOps Repo
    - git add deployment.yaml
    - git commit -m "🔥 EMERGENCY ROLLBACK: Reverted ${APP_NAME} to ${ROLLBACK_TARGET_TAG} [skip ci]"
    - git push origin main
    
    # 4. Trigger ArgoCD Instant Hard Refresh & Sync (Bypassing 3-minute poll interval)
    - echo "Triggering immediate ArgoCD Sync via REST API..."
    - >-
      curl -s -k -X POST
      -H "Authorization: Bearer ${ARGOCD_AUTH_TOKEN}"
      -H "Content-Type: application/json"
      https://${ARGOCD_SERVER}/api/v1/applications/${APP_NAME}-prod/sync
  rules:
    # Production ပေါ်တွင် Manual ဖြင့်သာ အရေးပေါ် run ခွင့်ပြုမည်
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual

# ------------------------------------------------------------------------------
# Post-Rollback Automated Health Verification Probe
# ------------------------------------------------------------------------------
verify:production_health:
  stage: health-verification
  image: curlimages/curl:latest
  needs:
    - job: emergency:rollback
  script:
    - echo "Verifying Production Health Endpoint..."
    - |
      for i in $(seq 1 12); do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.company.com/healthz || true)
        if [ "$STATUS" -eq 200 ]; then
          echo "✅ Production Service is Healthy (HTTP 200 OK)!"
          exit 0
        fi
        echo "Waiting for pods to roll over... Current status: $STATUS (Attempt $i/12)"
        sleep 5
      done
      echo "❌ Health check failed after rollback! Escalating to On-Call PagerDuty."
      exit 1
```

---

## 4. Rollback Speed & Metrics Comparison

| Incident Parameter | Manual Hotfix via Re-compile | Manual `kubectl rollout undo` | Automated GitOps Rollback |
| :--- | :--- | :--- | :--- |
| **Recovery Time (MTTR)** | 18 - 25 မိနစ် | ❌ FAILED (ArgoCD Overwrite) | **< 15 စက္ကန့်** |
| **Git History Tracking** | ⚠️ ရှုပ်ထွေးသည် | ❌ No Git Record (Drift) | **✅ Clean Commit History** |
| **Audit Compliance** | ❌ Manual SSH Login | ❌ Bypassed Governance | **✅ 100% Audited in GitLab & ArgoCD** |
| **Human Error Risk** | အလွန်မြင့်မားသည် | မြင့်မားသည် | **Zero Human Error (Automated CLI)** |

> [!TIP]
> **ArgoCD Self-Healing Best Practice:**
> Production Application များတွင် ArgoCD `syncPolicy` ကို အောက်ပါအတိုင်း အမြဲ configure လုပ်ထားပါ:
> ```yaml
> syncPolicy:
>   automated:
>     prune: true
>     selfHeal: true # Cluster ပေါ်တွင် တစ်စုံတစ်ယောက်မှ manual edit လုပ်ပါက Git အတိုင်း auto ပြန်ပြင်ပေးမည်
> ```
> ထို့ကြောင့် မည်သည့်အပြောင်းအလဲမဆို Git Commit အဖြစ်သာ ဝင်ရောက်လာစေခြင်းဖြင့် System Integrity ကို အာမခံနိုင်ပါသည်။
