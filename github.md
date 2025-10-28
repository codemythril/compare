class Script {
  process_incoming_request({ request }) {
    const data = request.content;
    
    let text = '';
    let icon = ':github:';
    
    // Push Event
    if (data.pusher) {
      text = `🔨 **${data.pusher.name}** pushed to \`${data.repository.name}\`\n`;
      text += `Branch: \`${data.ref.split('/').pop()}\`\n`;
      text += `Commits: ${data.commits.length}\n`;
      text += `[View Changes](${data.compare})`;
    }
    
    // Pull Request Event
    else if (data.pull_request) {
      const pr = data.pull_request;
      text = `📬 **${data.action}** pull request: [#${pr.number} ${pr.title}](${pr.html_url})\n`;
      text += `by ${pr.user.login}`;
    }
    
    // Issue Event
    else if (data.issue) {
      const issue = data.issue;
      text = `🐛 **${data.action}** issue: [#${issue.number} ${issue.title}](${issue.html_url})\n`;
      text += `by ${issue.user.login}`;
    }
    
    return {
      content: {
        text: text,
        icon_emoji: icon
      }
    };
  }
}


class Script {
  process_incoming_request({ request }) {
    const data = request.content;
    
    // Kein Payload vorhanden
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

    // ========== PUSH EVENT ==========
    if (data.pusher && data.commits) {
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

      // Zeige letzte 3 Commits
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
          },
          {
            title: 'Base',
            value: `\`${pr.base.ref}\``,
            short: true
          },
          {
            title: 'Head',
            value: `\`${pr.head.ref}\``,
            short: true
          }
        ],
        text: pr.body || '_No description_'
      });
    }

    // ========== ISSUE EVENT ==========
    else if (data.issue) {
      const issue = data.issue;
      const action = data.action;
      
      text = `🐛 Issue **${action}**`;
      
      let issueColor = '#6cc644';
      if (action === 'closed') issueColor = '#cb2431';
      
      attachments.push({
        color: issueColor,
        title: `#${issue.number}: ${issue.title}`,
        title_link: issue.html_url,
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Author',
            value: issue.user.login,
            short: true
          },
          {
            title: 'State',
            value: issue.state,
            short: true
          },
          {
            title: 'Labels',
            value: issue.labels.map(l => l.name).join(', ') || 'None',
            short: true
          }
        ],
        text: issue.body || '_No description_'
      });
    }

    // ========== RELEASE EVENT ==========
    else if (data.release) {
      const release = data.release;
      text = `🚀 **New Release**: ${release.tag_name}`;
      
      attachments.push({
        color: '#0366d6',
        title: release.name || release.tag_name,
        title_link: release.html_url,
        fields: [
          {
            title: 'Repository',
            value: `[${data.repository.full_name}](${data.repository.html_url})`,
            short: true
          },
          {
            title: 'Author',
            value: release.author.login,
            short: true
          }
        ],
        text: release.body || '_No release notes_'
      });
    }

    // ========== STAR EVENT ==========
    else if (data.action === 'started' && data.starred_at) {
      text = `⭐ **New Star** on ${data.repository.full_name}`;
      attachments.push({
        color: '#f9d71c',
        text: `${data.sender.login} starred the repository`
      });
    }

    // ========== FALLBACK für andere Events ==========
    else {
      text = `📦 GitHub Event: **${data.action || 'unknown'}**`;
      attachments.push({
        color: '#gray',
        text: `Repository: [${data.repository?.full_name || 'unknown'}](${data.repository?.html_url || '#'})\nEvent Type: ${request.headers['x-github-event'] || 'unknown'}`
      });
    }

    return {
      content: {
        username: 'GitHub',
        icon_emoji: emoji,
        text: text,
        attachments: attachments
      }
    };
  }
}
