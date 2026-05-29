# AISecOps - Agent agentic sur les actions de sécurité opérationnelle

Projet de recherche sur les modèles d'IA agentiques appliqués à la sécurité opérationnelle (OpSec).
Sujet : Quelle est la pertinence de la mise en place d'un modèle d'IA de type agentique sur les actions de sécurité opérationnelle (OpSec) dans un environnement complexe

<table align="center">
    <tr>
      <th>Author</th>
      <th>Author</th>
    </tr>
    <tr>
      <td align="center">
        <a href="https://github.com/nathanmartel21">
          <img src="https://github.com/nathanmartel21.png?size=115" width="115" alt="@nathanmartel21" /><br />
          <sub>@nathanmartel21</sub>
        </a>
        <br /><br />
        <a href="https://github.com/sponsors/nathanmartel21">
          <img src="https://img.shields.io/badge/sponsor-30363D?style=for-the-badge&logo=GitHub-Sponsors&logoColor=white" alt="Sponsor nathanmartel21" />
        </a>
      </td>
      <td align="center">
        <a href="https://github.com/Djegger">
          <img src="https://github.com/Djegger.png?size=115" width="115" alt="@Djegger" /><br />
          <sub>@Djegger</sub>
        </a>
        <br /><br />
        <a href="https://github.com/sponsors/Djegger">
          <img src="https://img.shields.io/badge/sponsor-30363D?style=for-the-badge&logo=GitHub-Sponsors&logoColor=white" alt="Sponsor Djegger" />
        </a>
      </td>
    </tr>
  </table>
</div>

---

## Stack technique :

| Composant | Rôle | Version |
|---|---|---|
| [kind](https://kind.sigs.k8s.io/) | Cluster Kubernetes local (dans Docker) | v0.23.0 |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Client CLI Kubernetes | v1.36.1 |
| [ArgoCD](https://argo-cd.readthedocs.io/) | Opérateur GitOps — synchronise Git → cluster | v2.11.0 |
| [smolagents](https://github.com/huggingface/smolagents) | Framework agent LLM (HuggingFace) | latest |
| Nvidia NIM | LLM inference (gpt-oss-120b) | API cloud |
| GitHub | Hébergement des dépôts Git + PR | — |

**Types d'attaque supportés :** `brute-force`, `ddos`, `scan`, `injection`, `exfiltration`

Si l'agent estime que la situation est ambiguë ou que le risque de régression est trop élevé, il retourne `ACTION: ALERT_ONLY` sans créer de PR — c'est l'humain qui décide de la suite.

---

## Prise en main complète (from scratch)

> Suivre ces étapes dans l'ordre.

```bash
mkdir AISecOps && cd AISecOps
git clone git@github.com:AISecOpsLab/openshift-infra.git
git clone git@github.com:AISecOpsLab/agentic-secops-engine.git
git clone git@github.com:AISecOpsLab/cmdb.git
git clone git@github.com:AISecOpsLab/openshift-configs.git
git clone git@github.com:AISecOpsLab/conf-serveurs.git
```

```bash
cp .env.example .env
```

```bash
mkdir -p ~/.local/bin
# Installer kind (kube in docker) + kubectl
curl -Lo ~/.local/bin/kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ~/.local/bin/kind
KUBECTL_VERSION=$(curl -sL https://dl.k8s.io/release/stable.txt)
curl -Lo ~/.local/bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
chmod +x ~/.local/bin/kubectl
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
kind version
kubectl version --client --short 2>/dev/null || kubectl version --client
```

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r agentic-secops-engine/requirements.txt```
```

créer le cluster kubernetes (kind) :

```bash
bash openshift-infra/setup-kind.sh
```
```bash
kubectl get nodes
```

installer ArgoCD

```bash
bash openshift-infra/install-argocd.sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d #mdp admin
```



```bash
kubectl get pods -n argocd
kubectl get application aisecops -n argocd -o wide
```

---

config les credentials GitHub dans ArgoCD

ArgoCD doit pouvoir lire le repo `openshift-configs` pour détecter les changements.

```bash
GITHUB_TOKEN=$(grep '^GITHUB_TOKEN=' .env | cut -d= -f2- | tr -d '"')

kubectl create secret generic argocd-repo-openshift-configs \
  -n argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/AISecOpsLab/openshift-configs.git \
  --from-literal=username=git \
  --from-literal=password="$GITHUB_TOKEN"

kubectl label secret argocd-repo-openshift-configs \
  -n argocd \
  argocd.argoproj.io/secret-type=repository

kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd --timeout=60s
```

forcer argo à vérifier le repo immédiatement :

```bash
kubectl annotate application aisecops -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite
sleep 15
kubectl get application aisecops -n argocd -o wide
```

UI argo

Dans un terminal séparé :

```bash
kubectl port-forward svc/argocd-server -n argocd 8888:443
# login admin, mdp par la commande au dessus
# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Tester pipeline : 

lancer une alerte de sécurité :

```bash
python3 agentic-secops-engine/run_agent.py '{
  "type": "brute-force",
  "source_ip": "198.51.100.42",
  "target_host": "srv-web-01",
  "port": 22,
  "protocol": "ssh",
  "timestamp": "2026-05-29T18:45:00Z",
  "severity": "high",
  "details": "487 tentatives de connexion SSH en 90 secondes sur port 22"
}'
```

L'agent va :
1. Lire la CMDB et le ConfigMap SSH de `srv-web-01`
2. Identifier la remédiation (réduire `MaxAuthTries`)
3. Créer une branche `fix/brute-force-srv-web-01-YYYYMMDD`
4. Modifier le ConfigMap et le pousser sur GitHub
5. Ouvrir une Pull Request