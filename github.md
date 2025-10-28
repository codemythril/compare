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
