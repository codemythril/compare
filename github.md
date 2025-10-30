class Script {
  process_incoming_request({ request }) {
    const data = request.content;
    const eventType = request.headers['x-github-event'];
    
    if (!data) {
      return {
        content: {
          text: '⚠️ Leeres Payload empfangen'
        }
      };
    }

    let text = '';
    let color = '#6cc644';
    let emoji = ':github:';
    let attachments = [];

    // ========== WORKFLOW RUN (Actions Pipeline) ==========
    if (eventType === 'workflow_run' || data.workflow_run) {
      const workflow = data.workflow_run;
      const conclusion = workflow.conclusion;
      const status = workflow.status;
      
      // Status-abhängige Farben und Emojis
      if (status === 'completed') {
        if (conclusion === 'success') {
          text = `✅ **Workflow erfolgreich**: ${workflow.name}`;
          color = '#28a745';
        } else if (conclusion === 'failure') {
          text = `❌ **Workflow fehlgeschlagen**: ${workflow.name}`;
          color = '#d73a49';
        } else if (conclusion === 'cancelled') {
          text = `⚠️ **Workflow abgebrochen**: ${workflow.name}`;
          color = '#ffc107';
        }
      } else if (status === 'in_progress') {
        text = `🔄 **Workflow läuft**: ${workflow.name}`;
        color = '#0366d6';
      } else if (status === 'queued') {
        text = `⏳ **Workflow in Warteschlange**: ${workflow.name}`;
        color = '#959da5';
      }

      attachments.push({
        color: color,
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Branch',
            value: `\`${workflow.head_branch}\``,
            short: true
          },
          {
            title: 'Status',
            value: status,
            short: true
          },
          {
            title: 'Conclusion',
            value: conclusion || 'N/A',
            short: true
          },
          {
            title: 'Triggered by',
            value: workflow.actor?.login || 'unknown',
            short: true
          },
          {
            title: 'Run Number',
            value: `#${workflow.run_number}`,
            short: true
          }
        ],
        actions: [
          {
            type: 'button',
            text: '🔗 View Workflow',
            url: workflow.html_url
          }
        ]
      });
    }

    // ========== WORKFLOW JOB (einzelner Job in Pipeline) ==========
    else if (eventType === 'workflow_job' || data.workflow_job) {
      const job = data.workflow_job;
      const conclusion = job.conclusion;
      const status = job.status;
      
      // Status-abhängige Darstellung
      if (status === 'completed') {
        if (conclusion === 'success') {
          text = `✅ **Job erfolgreich**: ${job.name}`;
          color = '#28a745';
        } else if (conclusion === 'failure') {
          text = `❌ **Job fehlgeschlagen**: ${job.name}`;
          color = '#d73a49';
        } else if (conclusion === 'cancelled') {
          text = `⚠️ **Job abgebrochen**: ${job.name}`;
          color = '#ffc107';
        }
      } else if (status === 'in_progress') {
        text = `🔄 **Job läuft**: ${job.name}`;
        color = '#0366d6';
      } else if (status === 'queued') {
        text = `⏳ **Job in Warteschlange**: ${job.name}`;
        color = '#959da5';
      }

      // Zeige Steps wenn vorhanden
      let stepsText = '';
      if (job.steps && job.steps.length > 0) {
        stepsText = '\n**Steps:**\n';
        job.steps.forEach(step => {
          let stepEmoji = '⚪';
          if (step.conclusion === 'success') stepEmoji = '✅';
          else if (step.conclusion === 'failure') stepEmoji = '❌';
          else if (step.status === 'in_progress') stepEmoji = '🔄';
          
          stepsText += `${stepEmoji} ${step.name}\n`;
        });
      }

      attachments.push({
        color: color,
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Status',
            value: status,
            short: true
          },
          {
            title: 'Conclusion',
            value: conclusion || 'N/A',
            short: true
          },
          {
            title: 'Runner',
            value: job.runner_name || 'N/A',
            short: true
          }
        ],
        text: stepsText,
        actions: [
          {
            type: 'button',
            text: '🔗 View Job',
            url: job.html_url
          }
        ]
      });
    }

    // ========== CHECK RUN (CI/CD Checks) ==========
    else if (eventType === 'check_run' || data.check_run) {
      const check = data.check_run;
      const conclusion = check.conclusion;
      const status = check.status;
      
      if (status === 'completed') {
        if (conclusion === 'success') {
          text = `✅ **Check erfolgreich**: ${check.name}`;
          color = '#28a745';
        } else if (conclusion === 'failure') {
          text = `❌ **Check fehlgeschlagen**: ${check.name}`;
          color = '#d73a49';
        } else if (conclusion === 'neutral') {
          text = `⚪ **Check neutral**: ${check.name}`;
          color = '#959da5';
        }
      } else if (status === 'in_progress') {
        text = `🔄 **Check läuft**: ${check.name}`;
        color = '#0366d6';
      }

      attachments.push({
        color: color,
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Status',
            value: status,
            short: true
          },
          {
            title: 'Conclusion',
            value: conclusion || 'N/A',
            short: true
          },
          {
            title: 'Started',
            value: new Date(check.started_at).toLocaleString('de-DE'),
            short: true
          }
        ],
        text: check.output?.summary || '',
        actions: [
          {
            type: 'button',
            text: '🔗 View Check',
            url: check.html_url
          }
        ]
      });
    }

    // ========== PUSH EVENT ==========
    else if (data.pusher && data.commits) {
      const repo = data.repository.name;
      const branch = data.ref.split('/').pop();
      const pusher = data.pusher.name;
      const commits = data.commits.length;
      
      text = `🔨 **Push to ${repo}**`;
      
      attachments.push({
        color: '#6cc644',
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Branch',
            value: `\`${branch}\``,
            short: true
          },
          {
            title: 'Pusher',
            value: pusher,
            short: true
          },
          {
            title: 'Commits',
            value: commits.toString(),
            short: true
          }
        ]
      });

      data.commits.slice(0, 3).forEach(commit => {
        attachments.push({
          color: '#gray',
          text: `[\`${commit.id.substring(0, 7)}\`](${commit.url}) ${commit.message}\n👤 ${commit.author.name}`,
          mrkdwn_in: ['text']
        });
      });

      if (data.compare) {
        attachments.push({
          actions: [
            {
              type: 'button',
              text: '🔗 View Changes',
              url: data.compare
            }
          ]
        });
      }
    }

    // ========== PULL REQUEST EVENT ==========
    else if (data.pull_request) {
      const pr = data.pull_request;
      const action = data.action;
      
      text = `📬 Pull Request **${action}**`;
      
      let prColor = '#6cc644';
      if (action === 'closed' && pr.merged) {
        prColor = '#6f42c1';
        text = `🎉 Pull Request **merged**`;
      } else if (action === 'closed') {
        prColor = '#cb2431';
      }

      attachments.push({
        color: prColor,
        title: `#${pr.number}: ${pr.title}`,
        title_link: pr.html_url,
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Author',
            value: pr.user.login,
            short: true
          }
        ]
      });
    }

    // ========== FALLBACK ==========
    else {
      text = `📦 **GitHub Event**: ${eventType || data.action || 'unknown'}`;
      attachments.push({
        color: '#959da5',
        text: `Repository: [${data.repository?.full_name || 'unknown'}](${data.repository?.html_url || '#'})\nEvent Type: \`${eventType || 'unknown'}\``
      });
    }

    return {
      content: {
        username: 'GitHub Actions',
        icon_emoji: emoji,
        text: text,
        attachments: attachments
      }
    };
  }
}



class Script {
  process_incoming_request({ request }) {
    const data = request.content;
    const eventType = request.headers['x-github-event'];
    
    if (!data) {
      return {
        content: {
          text: '⚠️ Leeres Payload empfangen'
        }
      };
    }

    let text = '';
    let color = '#6cc644';
    let emoji = ':github:';
    let attachments = [];

    // ========== WORKFLOW RUN (Actions Pipeline) ==========
    if (eventType === 'workflow_run' || data.workflow_run) {
      const workflow = data.workflow_run;
      const conclusion = workflow.conclusion;
      const status = workflow.status;
      const runId = workflow.id;
      const runNumber = workflow.run_number;
      const workflowFile = workflow.path.split('/').pop(); // z.B. main.yaml
      
      // Eindeutige Identifikation
      const buildIdentifier = `🏗️ **Build #${runNumber}** | Run ID: \`${runId}\``;
      
      // Status-abhängige Farben und Emojis
      if (status === 'completed') {
        if (conclusion === 'success') {
          text = `✅ ${buildIdentifier}\n**Workflow erfolgreich**: \`${workflowFile}\``;
          color = '#28a745';
          emoji = ':white_check_mark:';
        } else if (conclusion === 'failure') {
          text = `❌ ${buildIdentifier}\n**Workflow fehlgeschlagen**: \`${workflowFile}\``;
          color = '#d73a49';
          emoji = ':x:';
        } else if (conclusion === 'cancelled') {
          text = `⚠️ ${buildIdentifier}\n**Workflow abgebrochen**: \`${workflowFile}\``;
          color = '#ffc107';
          emoji = ':warning:';
        } else if (conclusion === 'skipped') {
          text = `⏭️ ${buildIdentifier}\n**Workflow übersprungen**: \`${workflowFile}\``;
          color = '#959da5';
          emoji = ':fast_forward:';
        }
      } else if (status === 'in_progress') {
        text = `🔄 ${buildIdentifier}\n**Workflow läuft**: \`${workflowFile}\``;
        color = '#0366d6';
        emoji = ':arrows_counterclockwise:';
      } else if (status === 'queued') {
        text = `⏳ ${buildIdentifier}\n**Workflow in Warteschlange**: \`${workflowFile}\``;
        color = '#959da5';
        emoji = ':hourglass:';
      }

      // Zeitstempel
      const startedAt = new Date(workflow.created_at).toLocaleString('de-DE');
      const duration = workflow.updated_at && workflow.created_at 
        ? Math.round((new Date(workflow.updated_at) - new Date(workflow.created_at)) / 1000) 
        : null;

      attachments.push({
        color: color,
        title: `${workflow.name} - ${workflowFile}`,
        title_link: workflow.html_url,
        fields: [
          {
            title: '📦 Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: '🌿 Branch',
            value: `\`${workflow.head_branch}\``,
            short: true
          },
          {
            title: '📊 Status',
            value: status.toUpperCase(),
            short: true
          },
          {
            title: '🎯 Ergebnis',
            value: conclusion ? conclusion.toUpperCase() : 'N/A',
            short: true
          },
          {
            title: '👤 Triggered by',
            value: workflow.triggering_actor?.login || workflow.actor?.login || 'unknown',
            short: true
          },
          {
            title: '🔢 Run Number',
            value: `#${runNumber}`,
            short: true
          },
          {
            title: '⏰ Gestartet',
            value: startedAt,
            short: true
          },
          {
            title: '⏱️ Dauer',
            value: duration ? `${duration}s` : 'läuft...',
            short: true
          },
          {
            title: '🆔 Run ID',
            value: `\`${runId}\``,
            short: false
          }
        ],
        footer: `Workflow-Datei: ${workflowFile} | Event: ${workflow.event}`,
        footer_icon: 'https://github.githubassets.com/favicons/favicon.png',
        ts: Math.floor(new Date(workflow.created_at).getTime() / 1000)
      });

      // Button zum Workflow
      attachments.push({
        actions: [
          {
            type: 'button',
            text: '🔗 Workflow anzeigen',
            url: workflow.html_url
          },
          {
            type: 'button',
            text: '📝 Logs anzeigen',
            url: `${workflow.html_url}/logs`
          }
        ]
      });
    }

    // ========== WORKFLOW JOB (einzelner Job in Pipeline) ==========
    else if (eventType === 'workflow_job' || data.workflow_job) {
      const job = data.workflow_job;
      const conclusion = job.conclusion;
      const status = job.status;
      const jobId = job.id;
      const runId = job.run_id;
      
      // Eindeutige Job-Identifikation
      const jobIdentifier = `🔧 **Job ID: \`${jobId}\`** | Run ID: \`${runId}\``;
      
      // Status-abhängige Darstellung
      if (status === 'completed') {
        if (conclusion === 'success') {
          text = `✅ ${jobIdentifier}\n**Job erfolgreich**: ${job.name}`;
          color = '#28a745';
        } else if (conclusion === 'failure') {
          text = `❌ ${jobIdentifier}\n**Job fehlgeschlagen**: ${job.name}`;
          color = '#d73a49';
        } else if (conclusion === 'cancelled') {
          text = `⚠️ ${jobIdentifier}\n**Job abgebrochen**: ${job.name}`;
          color = '#ffc107';
        }
      } else if (status === 'in_progress') {
        text = `🔄 ${jobIdentifier}\n**Job läuft**: ${job.name}`;
        color = '#0366d6';
      } else if (status === 'queued') {
        text = `⏳ ${jobIdentifier}\n**Job in Warteschlange**: ${job.name}`;
        color = '#959da5';
      }

      // Zeige Steps wenn vorhanden
      let stepsText = '';
      if (job.steps && job.steps.length > 0) {
        stepsText = '**Steps:**\n';
        job.steps.forEach((step, index) => {
          let stepEmoji = '⚪';
          if (step.conclusion === 'success') stepEmoji = '✅';
          else if (step.conclusion === 'failure') stepEmoji = '❌';
          else if (step.status === 'in_progress') stepEmoji = '🔄';
          else if (step.status === 'queued') stepEmoji = '⏳';
          
          const stepNumber = `[${index + 1}/${job.steps.length}]`;
          stepsText += `${stepEmoji} ${stepNumber} ${step.name}\n`;
        });
      }

      // Zeitberechnung
      const startedAt = job.started_at ? new Date(job.started_at).toLocaleString('de-DE') : 'N/A';
      const duration = job.completed_at && job.started_at
        ? Math.round((new Date(job.completed_at) - new Date(job.started_at)) / 1000)
        : null;

      attachments.push({
        color: color,
        title: job.name,
        title_link: job.html_url,
        fields: [
          {
            title: '📦 Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: '📊 Status',
            value: status.toUpperCase(),
            short: true
          },
          {
            title: '🎯 Ergebnis',
            value: conclusion ? conclusion.toUpperCase() : 'N/A',
            short: true
          },
          {
            title: '🖥️ Runner',
            value: job.runner_name || 'N/A',
            short: true
          },
          {
            title: '⏰ Gestartet',
            value: startedAt,
            short: true
          },
          {
            title: '⏱️ Dauer',
            value: duration ? `${duration}s` : 'läuft...',
            short: true
          },
          {
            title: '🆔 Job ID',
            value: `\`${jobId}\``,
            short: true
          },
          {
            title: '🏗️ Run ID',
            value: `\`${runId}\``,
            short: true
          }
        ],
        text: stepsText,
        footer: `Workflow: ${job.workflow_name}`,
        footer_icon: 'https://github.githubassets.com/favicons/favicon.png',
        ts: job.started_at ? Math.floor(new Date(job.started_at).getTime() / 1000) : null
      });

      // Button zum Job
      attachments.push({
        actions: [
          {
            type: 'button',
            text: '🔗 Job anzeigen',
            url: job.html_url
          }
        ]
      });
    }

    // ========== CHECK RUN (CI/CD Checks) ==========
    else if (eventType === 'check_run' || data.check_run) {
      const check = data.check_run;
      const conclusion = check.conclusion;
      const status = check.status;
      const checkId = check.id;
      
      const checkIdentifier = `🔍 **Check ID: \`${checkId}\`**`;
      
      if (status === 'completed') {
        if (conclusion === 'success') {
          text = `✅ ${checkIdentifier}\n**Check erfolgreich**: ${check.name}`;
          color = '#28a745';
        } else if (conclusion === 'failure') {
          text = `❌ ${checkIdentifier}\n**Check fehlgeschlagen**: ${check.name}`;
          color = '#d73a49';
        } else if (conclusion === 'neutral') {
          text = `⚪ ${checkIdentifier}\n**Check neutral**: ${check.name}`;
          color = '#959da5';
        }
      } else if (status === 'in_progress') {
        text = `🔄 ${checkIdentifier}\n**Check läuft**: ${check.name}`;
        color = '#0366d6';
      }

      const startedAt = check.started_at ? new Date(check.started_at).toLocaleString('de-DE') : 'N/A';

      attachments.push({
        color: color,
        title: check.name,
        title_link: check.html_url,
        fields: [
          {
            title: '📦 Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: '📊 Status',
            value: status.toUpperCase(),
            short: true
          },
          {
            title: '🎯 Ergebnis',
            value: conclusion ? conclusion.toUpperCase() : 'N/A',
            short: true
          },
          {
            title: '⏰ Gestartet',
            value: startedAt,
            short: true
          },
          {
            title: '🆔 Check ID',
            value: `\`${checkId}\``,
            short: false
          }
        ],
        text: check.output?.summary || '',
        footer: `Check Suite ID: ${check.check_suite?.id || 'N/A'}`,
        ts: check.started_at ? Math.floor(new Date(check.started_at).getTime() / 1000) : null
      });
    }

    // ========== PUSH EVENT ==========
    else if (data.pusher && data.commits) {
      const repo = data.repository.name;
      const branch = data.ref.split('/').pop();
      const pusher = data.pusher.name;
      const commits = data.commits.length;
      const commitSha = data.after.substring(0, 7);
      
      text = `🔨 **Push** | Commit: \`${commitSha}\`\n**Repository**: ${repo} | **Branch**: \`${branch}\``;
      
      attachments.push({
        color: '#6cc644',
        fields: [
          {
            title: '📦 Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: '🌿 Branch',
            value: `\`${branch}\``,
            short: true
          },
          {
            title: '👤 Pusher',
            value: pusher,
            short: true
          },
          {
            title: '📝 Commits',
            value: commits.toString(),
            short: true
          },
          {
            title: '🔖 Commit SHA',
            value: `\`${data.after}\``,
            short: false
          }
        ],
        footer: `Push Event`,
        ts: Math.floor(Date.now() / 1000)
      });

      data.commits.slice(0, 3).forEach(commit => {
        attachments.push({
          color: '#e1e4e8',
          text: `[\`${commit.id.substring(0, 7)}\`](${commit.url}) ${commit.message}\n👤 ${commit.author.name}`,
          mrkdwn_in: ['text']
        });
      });

      if (data.compare) {
        attachments.push({
          actions: [
            {
              type: 'button',
              text: '🔗 Änderungen anzeigen',
              url: data.compare
            }
          ]
        });
      }
    }

    // ========== FALLBACK ==========
    else {
      text = `📦 **GitHub Event**: ${eventType || data.action || 'unknown'}`;
      attachments.push({
        color: '#959da5',
        text: `Repository: [${data.repository?.full_name || 'unknown'}](${data.repository?.html_url || '#'})\nEvent Type: \`${eventType || 'unknown'}\``,
        ts: Math.floor(Date.now() / 1000)
      });
    }

    return {
      content: {
        username: 'GitHub Actions',
        icon_emoji: emoji,
        text: text,
        attachments: attachments
      }
    };
  }
}
