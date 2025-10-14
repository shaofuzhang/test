 const vscode = require('vscode');
const { PullRequestsProvider } = require('./src/pullRequestsProvider');
const { WelcomeProvider } = require('./src/welcomeProvider');
const { CNBApi } = require('./src/cnbApi');

// Webview面板管理器
class WebviewPanelManager {
  constructor() {
    this.panels = new Map(); // 用于存储打开的面板
  }

  /**
   * 获取或创建PR详情面板
   * @param {string} panelId - 面板ID (pr-${prId})
   * @param {string} title - 面板标题
   * @returns {vscode.WebviewPanel} 面板实例
   */
  getOrCreatePRDetailPanel(panelId, title) {
    // 检查是否已存在同ID的面板
    if (this.panels.has(panelId)) {
      const panel = this.panels.get(panelId);
      // 检查面板是否仍然有效
      if (panel && !panel.webview.isDisposed) {
        panel.reveal(vscode.ViewColumn.One);
        return panel;
      } else {
        // 面板已失效，从Map中移除
        this.panels.delete(panelId);
      }
    }

    // 创建新面板
    const panel = vscode.window.createWebviewPanel(
      'prDetail',
      title,
      vscode.ViewColumn.One,
      {
        enableScripts: true,
        retainContextWhenHidden: true
      }
    );

    // 监听面板关闭事件
    panel.onDidDispose(() => {
      this.panels.delete(panelId);
    }, null, []);

    // 存储面板引用
    this.panels.set(panelId, panel);
    return panel;
  }

  /**
   * 获取或创建文件差异面板
   * @param {string} panelId - 面板ID (diff-${prId}-${filePath})
   * @param {string} title - 面板标题
   * @returns {vscode.WebviewPanel} 面板实例
   */
  getOrCreateFileDiffPanel(panelId, title) {
    // 检查是否已存在同ID的面板
    if (this.panels.has(panelId)) {
      const panel = this.panels.get(panelId);
      // 检查面板是否仍然有效
      if (panel && !panel.webview.isDisposed) {
        panel.reveal(vscode.ViewColumn.One);
        return panel;
      } else {
        // 面板已失效，从Map中移除
        this.panels.delete(panelId);
      }
    }

    // 创建新面板
    const panel = vscode.window.createWebviewPanel(
      'fileDiff',
      title,
      vscode.ViewColumn.One,
      {
        enableScripts: true,
        retainContextWhenHidden: true
      }
    );

    // 监听面板关闭事件
    panel.onDidDispose(() => {
      this.panels.delete(panelId);
    }, null, []);

    // 存储面板引用
    this.panels.set(panelId, panel);
    return panel;
  }
}

/**
 * @param {vscode.ExtensionContext} context
 */
function activate(context) {
  // 初始化CNB API
  const cnbApi = new CNBApi();

  // 初始化Webview面板管理器
  const webviewPanelManager = new WebviewPanelManager();

  // 注册欢迎视图（当未配置令牌时）
  const welcomeProvider = new WelcomeProvider();
  vscode.window.registerTreeDataProvider('cnbWelcome', welcomeProvider);

  // 注册拉取请求视图
  const pullRequestsProvider = new PullRequestsProvider(cnbApi);
  vscode.window.registerTreeDataProvider('cnbPullRequests', pullRequestsProvider);

  // 注册命令
  const configureToken = vscode.commands.registerCommand('cnb.configureToken', async () => {
    const token = await vscode.window.showInputBox({
      prompt: '请输入您的CNB API令牌',
      password: true,
      validateInput: (value) => {
        if (!value || value.trim() === '') {
          return '令牌不能为空';
        }
        return null;
      }
    });

    if (token) {
      await context.secrets.store('cnb.token', token);
      await vscode.workspace.getConfiguration('cnb').update('token', token, vscode.ConfigurationTarget.Global);
      vscode.window.showInformationMessage('CNB令牌配置成功！');

      // 刷新视图
      welcomeProvider.refresh();
      pullRequestsProvider.refresh();
    }
  });

  const resetToken = vscode.commands.registerCommand('cnb.resetToken', async () => {
    const confirm = await vscode.window.showWarningMessage(
      '您确定要重置CNB令牌吗？这将删除当前令牌，您需要配置新令牌。',
      { modal: true },
      '是',
      '否'
    );

    if (confirm === '是') {
      // 从secrets和配置中清除令牌
      await context.secrets.delete('cnb.token');
      await vscode.workspace.getConfiguration('cnb').update('token', undefined, vscode.ConfigurationTarget.Global);

      vscode.window.showInformationMessage('CNB令牌已成功重置！');

      // 刷新视图以显示欢迎视图
      welcomeProvider.refresh();
      pullRequestsProvider.refresh();

      // 更新上下文以显示欢迎视图
      vscode.commands.executeCommand('setContext', 'cnb.token', false);
    }
  });

  const configureApiUrl = vscode.commands.registerCommand('cnb.configureApiUrl', async () => {
    const currentUrl = vscode.workspace.getConfiguration('cnb').get('apiUrl', 'https://api.cnb.cool');
    const url = await vscode.window.showInputBox({
      prompt: '请输入CNB API URL',
      value: currentUrl,
      validateInput: (value) => {
        if (!value || value.trim() === '') {
          return 'API URL不能为空';
        }
        // 验证URL格式
        if (!value.startsWith('http://') && !value.startsWith('https://')) {
          return 'API URL必须以http://或https://开头';
        }
        return null;
      }
    });

    if (url) {
      await vscode.workspace.getConfiguration('cnb').update('apiUrl', url, vscode.ConfigurationTarget.Global);
      vscode.window.showInformationMessage('CNB API URL配置成功！');

      // 刷新PR列表
      pullRequestsProvider.refresh();
    }
  });

  const refreshPullRequests = vscode.commands.registerCommand('cnb.refreshPullRequests', () => {
    pullRequestsProvider.refresh();
  });

  const mergePullRequest = vscode.commands.registerCommand('cnb.mergePullRequest', async (pullRequest) => {
    if (!pullRequest) {
      vscode.window.showErrorMessage('未选择拉取请求');
      return;
    }

    try {
      await cnbApi.mergePullRequest(pullRequest.id);
      vscode.window.showInformationMessage(`拉取请求 #${pullRequest.id} 已成功合并！`);
      // 刷新PR列表
      pullRequestsProvider.refresh();
    } catch (error) {
      vscode.window.showErrorMessage(`合并拉取请求失败: ${error.message}`);
    }
  });

  const declinePullRequest = vscode.commands.registerCommand('cnb.declinePullRequest', async (pullRequest) => {
    if (!pullRequest) {
      vscode.window.showErrorMessage('未选择拉取请求');
      return;
    }

    try {
      await cnbApi.declinePullRequest(pullRequest.id);
      vscode.window.showInformationMessage(`拉取请求 #${pullRequest.id} 已成功拒绝！`);
      // 刷新PR列表
      pullRequestsProvider.refresh();
    } catch (error) {
      vscode.window.showErrorMessage(`拒绝拉取请求失败: ${error.message}`);
    }
  });

  const viewPullRequest = vscode.commands.registerCommand('cnb.viewPullRequest', async (pullRequest) => {
    if (!pullRequest) {
      vscode.window.showErrorMessage('未选择拉取请求');
      return;
    }

    try {
      // 获取PR详情
      const prDetail = await cnbApi.getPullRequestDetail(pullRequest.id);

      // 创建PR详情文档（使用HTML格式）
      const content = `<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PR #${prDetail.id} - ${prDetail.title}</title>
    <style>
        :root {
            --vscode-editor-background: #1e1e1e;
            --vscode-editor-foreground: #d4d4d4;
            --vscode-sideBar-background: #252526;
            --vscode-sideBar-foreground: #ffffff;
            --vscode-button-background: #0e639c;
            --vscode-button-foreground: #ffffff;
            --vscode-button-hoverBackground: #1177bb;
            --vscode-errorForeground: #f48771;
            --vscode-descriptionForeground: #9e9e9e;
            --vscode-panel-border: #3c3c3c;
            --vscode-widget-shadow: rgba(0,0,0,0.3);
            --vscode-input-background: #3c3c3c;
            --vscode-input-foreground: #ffffff;
            --vscode-input-border: #3c3c3c;
            --vscode-focusBorder: #007fd4;
            --vscode-gitDecoration-addedResourceForeground: #81b88b;
            --vscode-gitDecoration-deletedResourceForeground: #c74e39;
            --vscode-gitDecoration-modifiedResourceForeground: #e2c08d;
            --vscode-statusBar-background: #007acc;
            --vscode-statusBar-foreground: #ffffff;
            --vscode-badge-background: #4d4d4d;
            --vscode-badge-foreground: #ffffff;
            --vscode-progressBar-background: #0e70c0;
        }
        
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
            margin: 0;
            padding: 0;
            background-color: var(--vscode-editor-background);
            color: var(--vscode-editor-foreground);
            transition: background-color 0.3s, color 0.3s;
        }
        
        .header {
            background: linear-gradient(135deg, var(--vscode-sideBar-background), #1a1a1a);
            color: var(--vscode-sideBar-foreground);
            padding: 25px 30px;
            box-shadow: 0 4px 12px var(--vscode-widget-shadow);
            position: relative;
            overflow: hidden;
        }
        
        .header::before {
            content: "";
            position: absolute;
            top: -50%;
            left: -50%;
            width: 200%;
            height: 200%;
            background: radial-gradient(circle, rgba(255,255,255,0.1) 0%, rgba(255,255,255,0) 70%);
            transform: rotate(30deg);
        }
        
        .header-content {
            position: relative;
            z-index: 1;
            max-width: 1200px;
            margin: 0 auto;
        }
        
        .header h1 {
            margin: 0 0 15px 0;
            font-size: 1.8em;
            font-weight: 600;
            display: flex;
            align-items: center;
            gap: 15px;
        }
        
        .pr-number {
            background: var(--vscode-badge-background);
            color: var(--vscode-badge-foreground);
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 0.8em;
            font-weight: 600;
        }
        
        .status {
            display: inline-block;
            padding: 6px 16px;
            border-radius: 20px;
            font-size: 0.9em;
            font-weight: 600;
            margin-left: 15px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        }
        
        .status.open {
            background: linear-gradient(135deg, #27ae60, #219653);
            color: white;
        }
        
        .status.merged {
            background: linear-gradient(135deg, #8e44ad, #7d3c98);
            color: white;
        }
        
        .status.declined {
            background: linear-gradient(135deg, #e74c3c, #c0392b);
            color: white;
        }
        
        .title {
            font-size: 1.6em;
            font-weight: 600;
            margin: 0 0 25px 0;
            padding: 20px 25px;
            background: var(--vscode-input-background);
            border-radius: 8px;
            box-shadow: 0 2px 8px var(--vscode-widget-shadow);
            border-left: 4px solid var(--vscode-focusBorder);
        }
        
        .content-container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 0 30px 30px 30px;
        }
        
        .info-panel {
            display: flex;
            background: var(--vscode-input-background);
            padding: 25px;
            border-radius: 8px;
            box-shadow: 0 4px 12px var(--vscode-widget-shadow);
            margin-bottom: 30px;
            gap: 30px;
        }
        
        .pr-info, .file-stats {
            flex: 1;
        }
        
        .section-title {
            font-size: 1.2em;
            font-weight: 600;
            margin: 0 0 20px 0;
            color: var(--vscode-editor-foreground);
            display: flex;
            align-items: center;
            gap: 10px;
        }
        
        .section-title::before {
            content: "";
            display: block;
            width: 4px;
            height: 20px;
            background: var(--vscode-focusBorder);
            border-radius: 2px;
        }
        
        .pr-details {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 20px;
        }
        
        .pr-field {
            display: flex;
            flex-direction: column;
        }
        
        .label {
            font-size: 0.9em;
            color: var(--vscode-descriptionForeground);
            margin-bottom: 8px;
            font-weight: 500;
            text-transform: uppercase;
            letter-spacing: 0.5px;
        }
        
        .value {
            font-size: 1.1em;
            font-weight: 500;
            padding: 12px 15px;
            background: rgba(255,255,255,0.05);
            border-radius: 6px;
            border: 1px solid var(--vscode-panel-border);
        }
        
        .branch-info {
            display: flex;
            align-items: center;
            gap: 15px;
            margin: 25px 0;
        }
        
        .branch {
            background: rgba(24, 144, 255, 0.1);
            color: var(--vscode-focusBorder);
            padding: 8px 16px;
            border-radius: 20px;
            font-weight: 500;
            font-size: 0.95em;
            border: 1px solid rgba(24, 144, 255, 0.3);
        }
        
        .arrow {
            color: var(--vscode-descriptionForeground);
            font-weight: bold;
            font-size: 1.2em;
        }
        
        .description {
            background: var(--vscode-input-background);
            border-radius: 8px;
            padding: 25px;
            box-shadow: 0 4px 12px var(--vscode-widget-shadow);
            margin-bottom: 30px;
        }
        
        .description-content {
            white-space: pre-wrap;
            line-height: 1.6;
        }
        
        .actions {
            background: var(--vscode-input-background);
            border-radius: 8px;
            padding: 25px;
            box-shadow: 0 4px 12px var(--vscode-widget-shadow);
            text-align: center;
        }
        
        .btn {
            padding: 12px 28px;
            border: none;
            border-radius: 6px;
            font-weight: 600;
            font-size: 1em;
            cursor: pointer;
            margin: 0 15px;
            transition: all 0.3s ease;
            box-shadow: 0 4px 8px rgba(0,0,0,0.2);
            min-width: 150px;
        }
        
        .btn-primary {
            background: linear-gradient(135deg, var(--vscode-button-background), #0a5a8c);
            color: var(--vscode-button-foreground);
        }
        
        .btn-primary:hover {
            background: linear-gradient(135deg, var(--vscode-button-hoverBackground), #0d6fb8);
            transform: translateY(-2px);
            box-shadow: 0 6px 12px rgba(0,0,0,0.3);
        }
        
        .btn-danger {
            background: linear-gradient(135deg, var(--vscode-errorForeground), #d77060);
            color: white;
        }
        
        .btn-danger:hover {
            background: linear-gradient(135deg, #ff9580, #f48771);
            transform: translateY(-2px);
            box-shadow: 0 6px 12px rgba(0,0,0,0.3);
        }
        
        .btn:active {
            transform: translateY(0);
        }
        
        .form-group {
            margin-bottom: 20px;
            text-align: left;
        }
        
        .form-group label {
            display: block;
            font-size: 0.9em;
            color: var(--vscode-descriptionForeground);
            margin-bottom: 8px;
            font-weight: 500;
            text-transform: uppercase;
            letter-spacing: 0.5px;
        }
        
        .form-control {
            width: 100%;
            padding: 12px 15px;
            background: rgba(255,255,255,0.05);
            border: 1px solid var(--vscode-panel-border);
            border-radius: 6px;
            color: var(--vscode-input-foreground);
            font-family: inherit;
            font-size: 1em;
            box-sizing: border-box;
        }
        
        .form-control:focus {
            outline: none;
            border-color: var(--vscode-focusBorder);
            box-shadow: 0 0 0 2px rgba(0, 127, 212, 0.3);
        }
        
        textarea.form-control {
            min-height: 100px;
            resize: vertical;
        }
        
        select.form-control {
            appearance: none;
            background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6 9 12 15 18 9'%3e%3c/polyline%3e%3c/svg%3e");
            background-repeat: no-repeat;
            background-position: right 15px center;
            background-size: 16px;
            padding-right: 40px;
        }
        
        .error-message {
            color: var(--vscode-errorForeground);
            font-size: 0.85em;
            margin-top: 6px;
            text-align: left;
        }
        
        .button-group {
            display: flex;
            gap: 15px;
            justify-content: center;
            margin-top: 10px;
        }
        
        @media (max-width: 768px) {
            .info-panel {
                flex-direction: column;
            }
            
            .header {
                padding: 20px;
            }
            
            .header h1 {
                font-size: 1.5em;
            }
            
            .title {
                font-size: 1.3em;
            }
            
            .content-container {
                padding: 0 20px 20px 20px;
            }
        }
    </style>
</head>
<body>
    <div class="header">
        <div class="header-content">
            <h1>
                PR 详情
                <span class="pr-number">#${prDetail.id}</span>
                <span class="status ${prDetail.status}">
                    ${prDetail.status === 'open' ? '开启中' : prDetail.status === 'merged' ? '已合并' : '已关闭'}
                </span>
            </h1>
        </div>
    </div>
    
    <div class="content-container">
        <div class="title">${prDetail.title}</div>
        
        <div class="info-panel">
            <div class="pr-info">
                <h2 class="section-title">基本信息</h2>
                <div class="pr-details">
                    <div class="pr-field">
                        <span class="label">作者</span>
                        <span class="value">${typeof prDetail.author === 'object' ? (prDetail.author.nickname || prDetail.author.login || '未知作者') : prDetail.author}</span>
                    </div>
                    <div class="pr-field">
                        <span class="label">创建时间</span>
                        <span class="value">${prDetail.createdAt ? new Date(prDetail.createdAt).toLocaleString() : '未知'}</span>
                    </div>
                    <div class="pr-field">
                        <span class="label">更新时间</span>
                        <span class="value">${prDetail.updatedAt ? new Date(prDetail.updatedAt).toLocaleString() : '未知'}</span>
                    </div>
                </div>
                
                <div class="branch-info">
                    <span class="label">分支信息</span>
                    <span class="branch">${prDetail.head.ref.replace('refs/heads/', '')}</span>
                    <span class="arrow">→</span>
                    <span class="branch">${prDetail.base.ref.replace('refs/heads/', '')}</span>
                </div>
            </div>
        </div>
        
        <div class="description">
            <h2 class="section-title">描述</h2>
            <div class="description-content">
                <p>${prDetail.description || '无描述'}</p>
            </div>
        </div>
        
        ${prDetail.status === 'open' ? `
        <div class="actions">
            <div class="form-group">
                <label for="mergeStyle">合并类型 *</label>
                <select id="mergeStyle" class="form-control">
                    <option value="merge" selected>直接合并</option>
                    <option value="squash">合并时压缩提交记录</option>
                    <option value="rebase">变基合并</option>
                </select>
            </div>
            
            <div class="form-group">
                <label for="commitTitle">提交标题 *</label>
                <input type="text" id="commitTitle" class="form-control" placeholder="输入提交标题" required>
                <div id="commitTitleError" class="error-message" style="display: none;">提交标题不能为空</div>
            </div>
            
            <div class="form-group">
                <label for="commitMessage">提交信息（可选）</label>
                <textarea id="commitMessage" class="form-control" placeholder="输入提交信息"></textarea>
            </div>
            
            <div class="button-group">
                <button class="btn btn-primary" onclick="mergePR()">合并请求</button>
                <button class="btn btn-danger" onclick="declinePR()">拒绝请求</button>
            </div>
        </div>
        ` : ''}
    </div>
    
    <script>
        // 监听主题变化消息
        window.addEventListener('message', event => {
            const message = event.data;
            switch (message.command) {
                case 'updateTheme':
                    // 更新CSS变量
                    const root = document.documentElement;
                    for (const [key, value] of Object.entries(message.theme)) {
                        root.style.setProperty(key, value);
                    }
                    break;
            }
        });
    </script>
</body>
<script>
    const vscode = acquireVsCodeApi();
    
    function mergePR() {
        // 获取表单值
        const mergeStyle = document.getElementById('mergeStyle').value;
        const commitTitle = document.getElementById('commitTitle').value;
        const commitMessage = document.getElementById('commitMessage').value;
        
        // 验证必填项
        const commitTitleError = document.getElementById('commitTitleError');
        if (!commitTitle.trim()) {
            commitTitleError.style.display = 'block';
            return;
        } else {
            commitTitleError.style.display = 'none';
        }
        
        vscode.postMessage({
            command: 'merge',
            prNumber: ${prDetail.id},
            mergeStyle: mergeStyle,
            commitTitle: commitTitle,
            commitMessage: commitMessage
        });
    }
    
    function declinePR() {
        vscode.postMessage({
            command: 'decline',
            prNumber: ${prDetail.id}
        });
    }
</script>
</html>`;

      // 创建Webview面板
      const panel = webviewPanelManager.getOrCreatePRDetailPanel(`pr-${prDetail.id}`, `PR #${prDetail.id} - ${prDetail.title}`);

      panel.webview.html = content;

      // 监听面板准备就绪事件
      panel.webview.onDidReceiveMessage(
        async (message) => {
          switch (message.command) {
            case 'merge':
              try {
                await cnbApi.mergePullRequest(message.prNumber, {
                  mergeStyle: message.mergeStyle,
                  commitTitle: message.commitTitle,
                  commitMessage: message.commitMessage
                });
                vscode.window.showInformationMessage(`拉取请求 #${message.prNumber} 已成功合并！`);
                panel.dispose();
                // 刷新PR列表
                pullRequestsProvider.refresh();
              } catch (error) {
                vscode.window.showErrorMessage(`合并拉取请求失败: ${error.message}`);
              }
              break;
            case 'decline':
              try {
                await cnbApi.declinePullRequest(message.prNumber, {
                  commitTitle: message.commitTitle,
                  commitMessage: message.commitMessage
                });
                vscode.window.showInformationMessage(`拉取请求 #${message.prNumber} 已成功拒绝！`);
                panel.dispose();
                // 刷新PR列表
                pullRequestsProvider.refresh();
              } catch (error) {
                vscode.window.showErrorMessage(`拒绝拉取请求失败: ${error.message}`);
              }
              break;
          }
        },
        undefined,
        context.subscriptions
      );

      // 监听主题变化并发送给Webview
      const themeChangeListener = vscode.window.onDidChangeActiveColorTheme((theme) => {
        const themeVariables = getThemeVariables(theme);
        panel.webview.postMessage({
          command: 'updateTheme',
          theme: themeVariables
        });
      });

      // 面板关闭时清理监听器
      panel.onDidDispose(() => {
        themeChangeListener.dispose();
      }, null, context.subscriptions);

      // 发送初始主题
      const initialTheme = getThemeVariables(vscode.window.activeColorTheme);
      panel.webview.postMessage({
        command: 'updateTheme',
        theme: initialTheme
      });

    } catch (error) {
      vscode.window.showErrorMessage(`加载拉取请求详情失败: ${error.message}`);
    }
  });

  // 注册文件点击命令
  const viewDiff = vscode.commands.registerCommand('cnb.viewDiff', async (fileItem) => {
    if (!fileItem) {
      vscode.window.showErrorMessage('未选择文件');
      return;
    }

    // 定义支持的文件扩展名
    const supportedExtensions = [
      // 文本文件
      '.txt', '.md', '.markdown', '.rst', '.log',
      // 编程语言文件
      '.js', '.ts', '.jsx', '.tsx', '.json', '.html', '.htm', '.css', '.scss', '.sass', '.less',
      '.java', '.py', '.pyw', '.cpp', '.c', '.h', '.hpp', '.cs', '.go', '.rs', '.rb', '.php', '.php3', '.php4', '.php5',
      '.swift', '.kt', '.kts', '.scala', '.pl', '.pm', '.t', '.sh', '.bash', '.zsh', '.fish',
      '.sql', '.pls', '.pck', '.pkb', '.pks', '.sqlj',
      '.xml', '.xsl', '.xslt', '.xsd', '.dtd', '.ent', '.mod',
      '.yml', '.yaml', '.toml', '.ini', '.cfg', '.conf', '.config',
      '.properties', '.env',
      // 配置文件
      '.gitignore', '.gitattributes', '.editorconfig', '.dockerignore',
      // 模板文件
      '.ejs', '.jinja', '.j2', '.handlebars', '.hbs', '.mustache',
      // 其他常见文本格式
      '.csv', '.tsv', '.diff', '.patch'
    ];

    // 定义不支持的二进制文件扩展名
    const unsupportedExtensions = [
      // 图片文件
      '.jpg', '.jpeg', '.png', '.gif', '.bmp', '.ico', '.svg', '.webp', '.tiff', '.tif',
      // 音频文件
      '.mp3', '.wav', '.ogg', '.flac', '.aac', '.wma', '.m4a',
      // 视频文件
      '.mp4', '.avi', '.mkv', '.mov', '.wmv', '.flv', '.webm',
      // 压缩文件
      '.zip', '.rar', '.7z', '.tar', '.gz', '.bz2', '.xz',
      // 可执行文件
      '.exe', '.msi', '.bat', '.cmd', '.dll', '.so', '.dylib',
      // 文档文件
      '.pdf', '.doc', '.docx', '.xls', '.xlsx', '.ppt', '.pptx',
      // 其他二进制文件
      '.bin', '.dat', '.db', '.sqlite', '.iso', '.dmg', '.pkg'
    ];

    // 检查文件扩展名是否不被支持
    const fileExtension = fileItem.filePath.substring(fileItem.filePath.lastIndexOf('.')).toLowerCase();
    const isUnsupported = unsupportedExtensions.includes(fileExtension);

    // 检查文件扩展名是否受支持
    const isSupported = supportedExtensions.includes(fileExtension) ||
      fileExtension === '' || // 没有扩展名的文件（如README, Makefile等）
      fileItem.filePath.indexOf('/') === -1; // 没有路径分隔符的文件

    // 对于没有扩展名的常见文件名，也认为是支持的
    const commonFiles = ['readme', 'makefile', 'dockerfile', 'license', 'changelog', 'contributing'];
    const fileName = fileItem.filePath.toLowerCase();
    const isCommonFile = commonFiles.some(commonFile => fileName.includes(commonFile));

    if (isUnsupported) {
      vscode.window.showWarningMessage(`暂不支持查看 "${fileItem.filePath}" 文件的差异。该文件是二进制文件或VS Code无法直接打开的文件类型。`);
      return;
    }

    if (!isSupported && !isCommonFile) {
      vscode.window.showWarningMessage(`暂不支持查看 "${fileItem.filePath}" 文件的差异。该文件类型可能为二进制文件或VS Code无法直接打开的文件类型。`);
      return;
    }

    try {
      // 获取文件差异
      const diffContent = await cnbApi.getFileDiff(fileItem.pullRequestId, fileItem.filePath);

      // 创建差异查看器Webview面板
      const panel = webviewPanelManager.getOrCreateFileDiffPanel(`diff-${fileItem.pullRequestId}-${fileItem.filePath}`, `Diff - ${fileItem.filePath}`);

      // 设置HTML内容，传递额外的PR和分支信息
      // 修改后的差异查看页面HTML代码
      panel.webview.html = `<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <title>Git Diff 分屏展示</title>
  <style>
    :root {
      --vscode-editor-background: #1e1e1e;
      --vscode-editor-foreground: #d4d4d4;
      --vscode-sideBar-background: #252526;
      --vscode-sideBar-foreground: #ffffff;
      --vscode-panel-border: #3c3c3c;
      --vscode-widget-shadow: rgba(0,0,0,0.3);
      --vscode-descriptionForeground: #9e9e9e;
      --vscode-gitDecoration-addedResourceForeground: #4ec9b0;
      --vscode-gitDecoration-deletedResourceForeground: #f48771;
      --vscode-gitDecoration-modifiedResourceForeground: #d7ba7d;
      --vscode-input-background: #3c3c3c;
      --vscode-input-foreground: #ffffff;
      --vscode-focusBorder: #007fd4;
      --vscode-badge-background: #4d4d4d;
      --vscode-badge-foreground: #ffffff;
    }
    
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
      margin: 0;
      padding: 0;
      background-color: var(--vscode-editor-background);
      color: var(--vscode-editor-foreground);
      transition: background-color 0.3s, color 0.3s;
    }
    
    .header {
      background: linear-gradient(135deg, var(--vscode-sideBar-background), #1a1a1a);
      color: var(--vscode-sideBar-foreground);
      padding: 25px 30px;
      box-shadow: 0 4px 12px var(--vscode-widget-shadow);
      position: relative;
      overflow: hidden;
    }
    
    .header::before {
      content: "";
      position: absolute;
      top: -50%;
      left: -50%;
      width: 200%;
      height: 200%;
      background: radial-gradient(circle, rgba(255,255,255,0.1) 0%, rgba(255,255,255,0) 70%);
      transform: rotate(30deg);
    }
    
    .header-content {
      position: relative;
      z-index: 1;
      max-width: 100%;
      margin: 0 auto;
    }
    
    .header h1 {
      margin: 0 0 15px 0;
      font-size: 1.8em;
      font-weight: 600;
      display: flex;
      align-items: center;
      gap: 15px;
    }
    
    .file-path {
      font-family: 'Monaco', 'Consolas', 'Courier New', monospace;
      font-size: 1em;
      background: var(--vscode-badge-background);
      color: var(--vscode-badge-foreground);
      padding: 6px 12px;
      border-radius: 20px;
      font-weight: 500;
    }
    
    .content-container {
      width: 100%;
      margin: 0;
      padding: 0;
    }
    
    .container {
      display: flex;
      height: calc(100vh - 200px);
      overflow: hidden;
      background: var(--vscode-input-background);
      border-radius: 8px;
      box-shadow: 0 4px 12px var(--vscode-widget-shadow);
    }
    
    .side {
      flex: 1;
      overflow-y: auto;
      box-sizing: border-box;
      display: flex;
      flex-direction: column;
    }
    
    .side-header {
      background: rgba(255,255,255,0.05);
      padding: 12px 15px;
      font-weight: 600;
      color: var(--vscode-editor-foreground);
      border-bottom: 1px solid var(--vscode-panel-border);
      font-size: 0.95em;
      text-align: center;
    }
    
    .side-content {
      flex: 1;
      overflow-y: auto;
    }
    
    .line {
      display: flex;
      min-height: 22px;
      padding: 6px 12px;
      border-bottom: 1px solid rgba(255,255,255,0.05);
      white-space: pre-wrap;
      font-family: 'Courier New', monospace;
      font-size: 0.95em;
    }
    
    .line-number {
      width: 50px;
      text-align: right;
      padding-right: 15px;
      color: #858585;
      font-weight: normal;
      user-select: none;
      font-size: 0.9em;
    }
    
    .content {
      flex: 1;
    }
    
    .deleted {
      background-color: rgba(244, 135, 113, 0.15);
      color: var(--vscode-gitDecoration-deletedResourceForeground);
    }
    
    .added {
      background-color: rgba(78, 201, 176, 0.15);
      color: var(--vscode-gitDecoration-addedResourceForeground);
    }
    
    .modified {
      background-color: rgba(215, 186, 125, 0.15);
      color: var(--vscode-gitDecoration-modifiedResourceForeground);
    }
    
    .unchanged {
      background-color: transparent;
      color: var(--vscode-editor-foreground);
    }
    
    .hunk {
      margin:  0;
      border: 0px solid var(--vscode-panel-border);
      border-radius: 4px;
      overflow: hidden;
    }
    
    .hunk-header {
      background: rgba(38, 79, 120, 0.3);
      padding: 0;
      font-size: 0.9em;
      color: var(--vscode-editor-foreground);
      font-family: 'Courier New', monospace;
      font-weight: 600;
    }
    
    @media (max-width: 768px) {
      .container {
        flex-direction: column;
        height: auto;
      }
      
      .header {
        padding: 20px;
      }
      
      .header h1 {
        font-size: 1.5em;
      }
      
      .content-container {
        padding: 0;
      }
    }
  </style>
</head>
<body>
  <div class="header">
    <div class="header-content">
      <h1>
        文件差异查看
        <span class="file-path">${fileItem.filePath}</span>
      </h1>
    </div>
  </div>
  
  <div class="content-container">
    <div class="container">
      <div class="side">
        <div class="side-content" id="leftSide"></div>
      </div>
      <div class="side">
        <div class="side-content" id="rightSide"></div>
      </div>
    </div>
  </div>
  
  <script>
    // 监听主题变化消息
    window.addEventListener('message', event => {
        const message = event.data;
        switch (message.command) {
            case 'updateTheme':
                // 更新CSS变量
                const root = document.documentElement;
                for (const [key, value] of Object.entries(message.theme)) {
                    root.style.setProperty(key, value);
                }
                break;
        }
    });
    
window.addEventListener('load', () => {
  const leftSide = document.getElementById('leftSide');
  const rightSide = document.getElementById('rightSide');
  const container = document.querySelector('.container');
  
  // 创建一个隐藏的测量元素来确定最大高度
  const measureElement = document.createElement('div');
  measureElement.style.visibility = 'hidden';
  measureElement.style.position = 'absolute';
  measureElement.style.width = '100%';
  document.body.appendChild(measureElement);
  
  // 同步两侧高度
  function syncHeights() {
    // 重置高度
    leftSide.style.height = 'auto';
    rightSide.style.height = 'auto';
    container.style.height = 'auto';
    
    // 获取内容高度
    const leftHeight = leftSide.scrollHeight;
    const rightHeight = rightSide.scrollHeight;
    const maxHeight = Math.max(leftHeight, rightHeight, 300); // 最小高度300px
    
    // 设置相同高度
    leftSide.style.height = maxHeight + 'px';
    rightSide.style.height = maxHeight + 'px';
    
    // 设置容器高度
    container.style.height = (maxHeight + 50) + 'px'; // 添加一些padding
  }
  
  // 同步滚动
  function syncScroll(source, target) {
    const scrollHandler = function() {
      if (!scrolling) {
        scrolling = true;
        target.scrollTop = source.scrollTop;
        scrolling = false;
      }
    };
    
    let scrolling = false;
    source.addEventListener('scroll', scrollHandler);
  }
  
  // 初始化同步
  syncHeights();
  syncScroll(leftSide, rightSide);
  syncScroll(rightSide, leftSide);
  
  // 监听内容变化
  const observer = new MutationObserver(syncHeights);
  observer.observe(leftSide, { childList: true, subtree: true });
  observer.observe(rightSide, { childList: true, subtree: true });
  
  // 窗口大小变化时重新同步
  window.addEventListener('resize', syncHeights);
});
  </script>
</body>

<script>
  // 获取文件内容和相关信息
  const fileContent = ${JSON.stringify(diffContent)};
  const fileInfo = {
    filePath: "${fileItem.filePath}",
    pullRequestId: "${fileItem.pullRequestId}"
  };
  
  // 解析 git diff 内容
  function parseDiff(content) {
    // 修复转义问题
    const lines = content.split('\\n');
    const hunks = [];
    let currentHunk = null;
    
    // 统计信息
    let additions = 0;
    let deletions = 0;
    let changes = 0;

    for (let i = 0; i < lines.length; i++) {
      const line = lines[i];

      // 检查是否是 hunk 头部
      if (line.startsWith('@@')) {
        // 如果有未处理的 hunk，先将其加入数组
        if (currentHunk) {
          hunks.push(currentHunk);
        }

        // 创建新的 hunk 对象
        currentHunk = {
          header: line,
          oldStart: 0,
          newStart: 0,
          oldLines: [],
          newLines: []
        };

        // 解析 hunk 头部的行号信息
        const hunkHeaderMatch = line.match(/@@ -(\\d+)(?:,(\\d+))? \\+(\\d+)(?:,(\\d+))? @@/);
        if (hunkHeaderMatch) {
          currentHunk.oldStart = parseInt(hunkHeaderMatch[1]);
          currentHunk.newStart = parseInt(hunkHeaderMatch[3]);
        }
      } else if (currentHunk && (line.startsWith('+') || line.startsWith('-') || line.startsWith(' '))) {
        // 只有在当前有 hunk 且行是有效的 diff 行时才处理
        if (line.startsWith('-')) {
          // 删除行
          currentHunk.oldLines.push({type: 'deleted', content: line});
          deletions++;
          changes++;
        } else if (line.startsWith('+')) {
          // 添加行
          currentHunk.newLines.push({type: 'added', content: line});
          additions++;
          changes++;
        } else if (line.startsWith(' ')) {
          // 不变行
          currentHunk.oldLines.push({type: 'unchanged', content: line});
          currentHunk.newLines.push({type: 'unchanged', content: line});
        }
      }
      // 忽略其他行（如文件头 --- a/... +++ b/...）
    }

    // 添加最后一个 hunk
    if (currentHunk) {
      hunks.push(currentHunk);
    }
    
    // 返回解析结果和统计信息
    return {
      hunks: hunks,
      stats: {
        additions: additions,
        deletions: deletions,
        changes: changes
      }
    };
  }

  // 渲染分屏
  function renderDiff(parsedDiff) {
    const leftSide = document.getElementById('leftSide');
    const rightSide = document.getElementById('rightSide');
    
    // 清空容器
    leftSide.innerHTML = '';
    rightSide.innerHTML = '';

    // 为每个 hunk 创建一个 div
    parsedDiff.hunks.forEach((hunk, hunkIndex) => {
      const hunkDivLeft = document.createElement('div');
      hunkDivLeft.className = 'hunk';
      const hunkDivRight = document.createElement('div');
      hunkDivRight.className = 'hunk';


      // 跟踪实际行号
      let oldLineNumber = hunk.oldStart;
      let newLineNumber = hunk.newStart;

      // 处理每一行
      const maxLines = Math.max(hunk.oldLines.length, hunk.newLines.length);
      for (let j = 0; j < maxLines; j++) {
        const oldLineObj = hunk.oldLines[j];
        const newLineObj = hunk.newLines[j];

        // 左侧：旧代码
        if (oldLineObj) {
          const lineLeft = document.createElement('div');
          lineLeft.className = 'line';

          // 行号
          const lineNumberLeft = document.createElement('div');
          lineNumberLeft.className = 'line-number';
          if (oldLineObj.type !== 'added') {
            lineNumberLeft.textContent = oldLineNumber;
            if (oldLineObj.type !== 'added') oldLineNumber++;
          }
          lineLeft.appendChild(lineNumberLeft);

          // 内容
          const contentLeft = document.createElement('div');
          contentLeft.className = 'content';
          contentLeft.textContent = oldLineObj.content;
          lineLeft.appendChild(contentLeft);

          // 设置样式
          if (oldLineObj.type === 'deleted') {
            lineLeft.classList.add('deleted');
            contentLeft.classList.add('deleted');
          } else if (oldLineObj.type === 'added') {
            lineLeft.classList.add('modified');
            contentLeft.classList.add('modified');
          } else if (oldLineObj.type === 'unchanged') {
            lineLeft.classList.add('unchanged');
            contentLeft.classList.add('unchanged');
          }

          hunkDivLeft.appendChild(lineLeft);
        }

        // 右侧：新代码
        if (newLineObj) {
          const lineRight = document.createElement('div');
          lineRight.className = 'line';

          // 行号
          const lineNumberRight = document.createElement('div');
          lineNumberRight.className = 'line-number';
          if (newLineObj.type !== 'deleted') {
            lineNumberRight.textContent = newLineNumber;
            if (newLineObj.type !== 'deleted') newLineNumber++;
          }
          lineRight.appendChild(lineNumberRight);

          // 内容
          const contentRight = document.createElement('div');
          contentRight.className = 'content';
          contentRight.textContent = newLineObj.content;
          lineRight.appendChild(contentRight);

          // 设置样式
          if (newLineObj.type === 'added') {
            lineRight.classList.add('added');
            contentRight.classList.add('added');
          } else if (newLineObj.type === 'deleted') {
            lineRight.classList.add('modified');
            contentRight.classList.add('modified');
          } else if (newLineObj.type === 'unchanged') {
            lineRight.classList.add('unchanged');
            contentRight.classList.add('unchanged');
          }

          hunkDivRight.appendChild(lineRight);
        }
      }

      // 将 hunk 添加到对应的容器中
      leftSide.appendChild(hunkDivLeft);
      rightSide.appendChild(hunkDivRight);
    });
  }

  // 解析并渲染
  const parsedDiff = parseDiff(fileContent);
  renderDiff(parsedDiff);
</script>
</body>
</html>`;

    } catch (error) {
      vscode.window.showErrorMessage(`加载文件差异失败: ${error.message}`);
    }
  });



  context.subscriptions.push(
    configureToken,
    resetToken,
    configureApiUrl,
    refreshPullRequests,
    mergePullRequest,
    declinePullRequest,
    viewPullRequest,
    viewDiff
  );

  // 检查令牌是否存在，如果不存在则显示欢迎视图
  const token = vscode.workspace.getConfiguration('cnb').get('token');
  if (!token) {
    vscode.commands.executeCommand('setContext', 'cnb.token', false);
  } else {
    vscode.commands.executeCommand('setContext', 'cnb.token', true);
  }
}

/**
 * 获取主题相关的CSS变量
 * @param {vscode.ColorTheme} theme VS Code主题对象
 * @returns {Object} 包含CSS变量的对象
 */
function getThemeVariables(theme) {
  // 根据主题类型设置基础颜色
  let baseColors = {};

  switch (theme.kind) {
    case vscode.ColorThemeKind.Light:
    case vscode.ColorThemeKind.HighContrastLight:
      // 浅色主题
      baseColors = {
        '--vscode-editor-background': '#ffffff',
        '--vscode-editor-foreground': '#000000',
        '--vscode-sideBar-background': '#f3f3f3',
        '--vscode-sideBar-foreground': '#000000',
        '--vscode-button-background': '#007acc',
        '--vscode-button-foreground': '#ffffff',
        '--vscode-button-hoverBackground': '#005a9e',
        '--vscode-errorForeground': '#e51400',
        '--vscode-descriptionForeground': '#717171',
        '--vscode-panel-border': '#cccccc',
        '--vscode-widget-shadow': 'rgba(0,0,0,0.1)',
        '--vscode-input-background': '#ffffff',
        '--vscode-input-foreground': '#000000',
        '--vscode-input-border': '#cecece',
        '--vscode-focusBorder': '#007fd4',
        '--vscode-gitDecoration-addedResourceForeground': '#487e02',
        '--vscode-gitDecoration-deletedResourceForeground': '#ad0707',
        '--vscode-gitDecoration-modifiedResourceForeground': '#895503',
        '--vscode-statusBar-background': '#007acc',
        '--vscode-statusBar-foreground': '#ffffff'
      };
      break;
    case vscode.ColorThemeKind.Dark:
    case vscode.ColorThemeKind.HighContrast:
    default:
      // 深色主题（默认）
      baseColors = {
        '--vscode-editor-background': '#1e1e1e',
        '--vscode-editor-foreground': '#d4d4d4',
        '--vscode-sideBar-background': '#252526',
        '--vscode-sideBar-foreground': '#ffffff',
        '--vscode-button-background': '#0e639c',
        '--vscode-button-foreground': '#ffffff',
        '--vscode-button-hoverBackground': '#1177bb',
        '--vscode-errorForeground': '#f48771',
        '--vscode-descriptionForeground': '#9e9e9e',
        '--vscode-panel-border': '#3c3c3c',
        '--vscode-widget-shadow': 'rgba(0,0,0,0.3)',
        '--vscode-input-background': '#3c3c3c',
        '--vscode-input-foreground': '#ffffff',
        '--vscode-input-border': '#3c3c3c',
        '--vscode-focusBorder': '#007fd4',
        '--vscode-gitDecoration-addedResourceForeground': '#4ec9b0',
        '--vscode-gitDecoration-deletedResourceForeground': '#f48771',
        '--vscode-gitDecoration-modifiedResourceForeground': '#d7ba7d',
        '--vscode-statusBar-background': '#007acc',
        '--vscode-statusBar-foreground': '#ffffff',
        '--vscode-badge-background': '#4d4d4d',
        '--vscode-badge-foreground': '#ffffff'
      };
      break;
  }

  return baseColors;
}

module.exports = {
  activate
};
