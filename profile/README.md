# AISecOps - IA agentique appliquée à la sécurité opérationnelle

Projet de recherche : **un agent LLM peut-il prendre en charge des actions de sécurité opérationnelle (SecOps) dans un environnement réaliste ?**

L'agent reçoit une alerte, lit le contexte de l'infra, et **propose** une correction sous forme de pull request. Il ne merge jamais, il ne touche jamais la prod directement : c'est un humain qui valide (HITL).

<table align="center">
  <tr><th>Author</th><th>Author</th></tr>
  <tr>
    <td align="center">
      <a href="https://github.com/nathanmartel21">
        <img src="https://github.com/nathanmartel21.png?size=115" width="115" alt="@nathanmartel21" /><br />
        <sub>@nathanmartel21</sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/Djegger">
        <img src="https://github.com/Djegger.png?size=115" width="115" alt="@Djegger" /><br />
        <sub>@Djegger</sub>
      </a>
    </td>
  </tr>
</table>

---

## Accès aux interfaces du lab

Se connecter au VPN de l'école avant bien évidemment

| Interface | Lien | Identifiants |
|---|---|---|
| **Console OpenShift** (pods, services, ingress) | https://console.159.31.247.120.nip.io:8443 | `openshift-infra/.console-auth` |
| **ArgoCD** (GitOps) | https://argocd.159.31.247.120.nip.io:8443 | `admin` + secret `argocd-initial-admin-secret` |
| **Wazuh** (SIEM) | https://wazuh.159.31.247.120.nip.io:8443 | `admin` + `WAZUH_INDEXER_PASSWORD` du `.env` |
| **srv-web-01** (cible des attaques) | https://web.159.31.247.120.nip.io:8443 | aucun - site public simulé |
| **NetBox** (CMDB) | http://localhost:8000 | local au VPS uniquement |

Les autres serveurs du lab (DNS, mail, AD, base de données, bastion...) n'ont pas d'interface web : ce sont des cibles simulées. On les inspecte depuis la console OpenShift, ou en ligne de commande :

```bash
kubectl -n aisecops get pods
kubectl -n aisecops logs deploy/srv-dns-01 -c bind9
```

Mot de passe ArgoCD :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Ou aller directement sur OKD (console) 

---

## Fonctionnement

Le **pont** (`wazuh-bridge`) interroge l'indexer Wazuh toutes les 15 s, filtre le bruit, regroupe les alertes d'un même attaquant, puis lance l'agent dans un conteneur.

L'agent a aujourd'hui **6 fonctions** au catalogue :

| Fonction | Ce qu'elle fait | Risque |
|---|---|---|
| `remediation-config` | corrige un ConfigMap après une alerte → PR | modification |
| `triage-alerte` | qualifie l'alerte (vrai/faux positif, escalade ?) | triage |
| `audit-conformite` | audite le durcissement → PR de mise en conformité | modification |
| `gestion-vuln` | rapport CVE priorisé (CMDB × CVSS) | lecture |
| `rapport-incident` | rapport post-incident (timeline, MITRE) | lecture |
| `hardening-proposer` | propose du durcissement sous forme de tickets | lecture |

Si la situation est ambiguë ou le risque trop élevé, l'agent répond `alert_only` sans rien modifier

---

## Stack

| Composant | Rôle | Version |
|---|---|---|
| [kind](https://kind.sigs.k8s.io/) | cluster Kubernetes local | v0.23.0 / k8s 1.30 |
| [ArgoCD](https://argo-cd.readthedocs.io/) | GitOps : Git → cluster | v2.11.0 |
| [Wazuh](https://wazuh.com/) | SIEM (détection) | 4.13.1 |
| [console OpenShift](https://github.com/openshift/console) | interface graphique du cluster | origin-console 4.15 |
| [NetBox](https://netbox.dev/) | CMDB (criticité, dépendances, sources de confiance) | v4.4 |
| [smolagents](https://github.com/huggingface/smolagents) | framework d'agent LLM | latest |
| [Ollama](https://ollama.com/) | inférence on-prem (agent + juge) | qwen3-30b / aisecops-judge |

---

<!-- ## Les dépôts

**Le cœur**
- [`agentic-secops-engine-interne`](https://github.com/AISecOpsLab/agentic-secops-engine-interne) - l'agent (LLM on-prem) + le juge
- [`agentic-secops-engine`](https://github.com/AISecOpsLab/agentic-secops-engine) - même agent, variante LLM cloud
- [`wazuh-bridge`](https://github.com/AISecOpsLab/wazuh-bridge) - le pont Wazuh → agent

**L'infra**
- [`openshift-infra`](https://github.com/AISecOpsLab/openshift-infra) - scripts d'installation (kind, ArgoCD, Wazuh, ingress, console)
- [`openshift-configs`](https://github.com/AISecOpsLab/openshift-configs) - dépôt GitOps, les ConfigMaps que l'agent modifie
- [`cmdb`](https://github.com/AISecOpsLab/cmdb) / [`cmdb-netbox`](https://github.com/AISecOpsLab/cmdb-netbox) - la CMDB
- [`conf-serveurs`](https://github.com/AISecOpsLab/conf-serveurs) - configs des serveurs simulés

**Les connaissances de l'agent**
- [`kb-remediation`](https://github.com/AISecOpsLab/kb-remediation) - playbooks (MITRE D3FEND/ATT&CK, OWASP)
- [`kb-cti`](https://github.com/AISecOpsLab/kb-cti) - enrichissement CVE

**L'évaluation**
- [`ait-dataset`](https://github.com/AISecOpsLab/ait-dataset) - bancs de mesure sur le dataset AIT
- [`judge-eval`](https://github.com/AISecOpsLab/judge-eval) - étalonnage du juge contre des annotations humaines
- [`test-attaque`](https://github.com/AISecOpsLab/test-attaque) - scénarios d'attaque et tests d'injection de prompt

**Archives** - [`test-code-agent`](https://github.com/AISecOpsLab/test-code-agent), [`test-tool-calling-agent`](https://github.com/AISecOpsLab/test-tool-calling-agent) : prototypes abandonnés, gardés pour documenter les choix. -->

<!-- ---

## Démarrer le lab

```bash
git clone git@github.com:AISecOpsLab/lab-bootstrap.git && bash lab-bootstrap/install.sh
cp .env.example .env        # puis remplir les clés
```

Puis, dans l'ordre :

```bash
bash openshift-infra/setup-kind.sh       # cluster
bash openshift-infra/install-argocd.sh   # GitOps
bash openshift-infra/install-wazuh.sh    # SIEM
bash openshift-infra/install-ingress.sh  # expose les UI
bash openshift-infra/install-console.sh  # console OpenShift
bash openshift-infra/install-bridge.sh   # le pont (conteneur permanent)
```

Le détail de chaque brique est dans le README du dépôt correspondant. -->

<!-- Lancer l'agent à la main, sans passer par le pont :

```bash
echo '{"fonction":"triage-alerte","type":"scan","source_ip":"203.0.113.45","target_host":"srv-web-01","details":"balayage de ports"}' \
  | python agentic-secops-engine-interne/run_agent.py
``` -->
