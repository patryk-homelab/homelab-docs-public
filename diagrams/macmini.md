# Mac mini — Service Diagram

Auto-generated from `HOMELAB-MACMINI.md`'s service inventory by `generate-push-macmini-diagram.sh`. Do not edit by hand - it is overwritten on every run and only committed when its content changes.

```mermaid
flowchart TD
  subgraph ai_llm["AI/LLM"]
    ollama["Ollama<br/>127.0.0.1:11434<br/>/opt/homebrew/bin/ollama"]
  end
  subgraph automation["Automation"]
    daily_homelab_report["Daily Homelab Report<br/>(no exposed port)<br/>/Users/patrykmac/homelab/daily-report/daily-homelab-report.sh"]
    homelab_backup["Homelab Backup<br/>(no exposed port)<br/>/Users/patrykmac/homelab/backup"]
    homelab_docs_public_github_push["Homelab-docs-public GitHub Push<br/>(no exposed port)<br/>/Users/patrykmac/homelab-docs-public-github"]
    homelab_repo_git_snapshot["Homelab-repo Git Snapshot<br/>(no exposed port)<br/>/Users/patrykmac/homelab-repo"]
    icloud_document_exchange["iCloud Document Exchange<br/>(no exposed port)<br/>/Users/patrykmac/Library/Mobile Documents/com~apple~CloudDocs/Desktop/HOMELAB"]
    mac_mini_service_diagram["Mac mini Service Diagram<br/>192.168.10.13<br/>/Users/patrykmac/homelab/backup/generate-push-macmini-diagram.sh"]
  end
  subgraph discord_bot["Discord Bot"]
    klefedroniarz_discord_bot["Klefedroniarz Discord Bot<br/>(no exposed port)<br/>/Users/patrykmac/homelab/discord-bot"]
  end
  subgraph media["Media"]
    metube["MeTube<br/>http://192.168.10.13:8091<br/>/Users/patrykmac/homelab/metube/app"]
    metube_cleanup["MeTube Cleanup<br/>(no exposed port)<br/>/Users/patrykmac/homelab/metube/MeTubeCleanupLauncher.app"]
  end
  subgraph monitoring["Monitoring"]
    glances["Glances<br/>http://192.168.10.13:61208/api/4/quicklook<br/>/opt/homebrew/bin/glances"]
    homelab_status_agent["Homelab Status Agent<br/>http://192.168.10.13:3005/status<br/>/Users/patrykmac/homelab/status-agent/server.py"]
    smartmontools_health_monitor["Smartmontools Health Monitor<br/>(no exposed port)<br/>/opt/homebrew/bin/smartctl"]
    ssd_mini_availability_monitor["SSD-MINI Availability Monitor<br/>(no exposed port)<br/>/Users/patrykmac/homelab/monitoring/check-ssd-mini.sh"]
    uptime_kuma["Uptime Kuma<br/>http://192.168.10.13:3003<br/>/Users/patrykmac/homelab/uptime-kuma/app"]
  end
  subgraph networking["Networking"]
    adguard_home["AdGuard Home<br/>http://192.168.10.13:3002"]
    upsnap["UpSnap<br/>http://192.168.10.13:8090<br/>/Users/patrykmac/homelab/upsnap/"]
  end
  subgraph storage["Storage"]
    filebrowser_quantum["FileBrowser Quantum<br/>http://192.168.10.13:8085/<br/>/Users/patrykmac/homelab/filebrowser/bin/filebrowser"]
  end
```
