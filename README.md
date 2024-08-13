网页内容
```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FRP 配置生成器</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .container { max-width: 800px; margin: auto; }
        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 5px; }
        .form-group input, .form-group select { width: 100%; padding: 8px; box-sizing: border-box; }
        button { padding: 10px 20px; background-color: #007bff; color: white; border: none; cursor: pointer; }
        button:disabled { background-color: #aaa; cursor: not-allowed; }
        pre { background-color: #f5f5f5; padding: 10px; border: 1px solid #ddd; overflow: auto; }
        .proxy-config { margin-top: 20px; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>FRP 配置生成器</h1>
        <p>使用此工具生成 FRP 服务器和客户端配置文件。填写下面的详细信息，然后点击“生成配置”以生成配置文件。您也可以复制生成的配置以便于使用。</p>
        
        <form id="configForm">
            <fieldset>
                <legend>[common]（服务器）</legend>
                <div class="form-group">
                    <label for="common_bind_addr">绑定地址：</label>
                    <input type="text" id="common_bind_addr" name="common_bind_addr" value="0.0.0.0">
                </div>
                <div class="form-group">
                    <label for="common_bind_port">绑定端口：</label>
                    <input type="number" id="common_bind_port" name="common_bind_port" value="7000">
                </div>
                <div class="form-group">
                    <label for="common_token">令牌：</label>
                    <input type="text" id="common_token" name="common_token" value="my_secure_token">
                </div>
                <div class="form-group">
                    <label for="common_log_level">日志级别：</label>
                    <select id="common_log_level" name="common_log_level">
                        <option value="debug" selected>debug</option>
                        <option value="info">info</option>
                        <option value="warn">warn</option>
                        <option value="error">error</option>
                    </select>
                </div>
                <div class="form-group">
                    <label for="common_log_file">日志文件：</label>
                    <input type="text" id="common_log_file" name="common_log_file" value="./frps.log">
                </div>
                <div class="form-group">
                    <label for="common_log_max_days">日志保存天数：</label>
                    <input type="number" id="common_log_max_days" name="common_log_max_days" value="7">
                </div>
            </fieldset>
            <fieldset>
                <legend>[dashboard]（服务器）</legend>
                <div class="form-group">
                    <label for="dashboard_enable">启用仪表板：</label>
                    <input type="checkbox" id="dashboard_enable" name="dashboard_enable" checked>
                </div>
                <div class="form-group">
                    <label for="dashboard_port">仪表板端口：</label>
                    <input type="number" id="dashboard_port" name="dashboard_port" value="7500">
                </div>
                <div class="form-group">
                    <label for="dashboard_user">仪表板用户：</label>
                    <input type="text" id="dashboard_user" name="dashboard_user" value="admin">
                </div>
                <div class="form-group">
                    <label for="dashboard_pwd">仪表板密码：</label>
                    <input type="text" id="dashboard_pwd" name="dashboard_pwd" value="admin">
                </div>
            </fieldset>
            <fieldset>
                <legend>代理配置</legend>
                <div id="proxyList"></div>
                <button type="button" onclick="addProxyConfig()">添加代理</button>
            </fieldset>
            <div class="form-group">
                <label for="server_addr">服务器地址：</label>
                <input type="text" id="server_addr" name="server_addr" value="127.0.0.1">
            </div>
            <button type="button" onclick="generateConfig()">生成配置</button>
            <button type="button" onclick="copyConfig()">复制配置</button>
        </form>
        
        <h2>生成的服务器配置 (frps.ini)</h2>
        <pre id="serverConfigOutput"></pre>
        <h2>生成的客户端配置 (frpc.ini)</h2>
        <pre id="clientConfigOutput"></pre>
    </div>

    <script>
        let proxyCount = 0;

        function generateFormGroup(id, label, value, type = 'text') {
            return `
                <div class="form-group">
                    <label for="${id}">${label}</label>
                    <input type="${type}" id="${id}" name="${id}" value="${value}">
                </div>
            `;
        }

        function generateSelectGroup(id, label, options, selected) {
            const optionsHtml = options.map(opt => `<option value="${opt}" ${opt === selected ? 'selected' : ''}>${opt}</option>`).join('');
            return `
                <div class="form-group">
                    <label for="${id}">${label}</label>
                    <select id="${id}" name="${id}">${optionsHtml}</select>
                </div>
            `;
        }

        function addProxyConfig() {
            proxyCount++;
            const proxyList = document.getElementById('proxyList');
            const proxyDiv = document.createElement('div');
            proxyDiv.classList.add('proxy-config');
            proxyDiv.innerHTML = `
                <fieldset>
                    <legend>代理 ${proxyCount}</legend>
                    ${generateFormGroup(`proxy_${proxyCount}_name`, '名称:', `proxy${proxyCount}`)}
                    ${generateSelectGroup(`proxy_${proxyCount}_type`, '类型:', ['http', 'tcp', 'udp'], 'http')}
                    ${generateFormGroup(`proxy_${proxyCount}_local_ip`, '本地 IP:', '127.0.0.1')}
                    ${generateFormGroup(`proxy_${proxyCount}_local_port`, '本地端口:', '80', 'number')}
                    ${generateFormGroup(`proxy_${proxyCount}_remote_port`, '远程端口:', '6000', 'number')}
                </fieldset>
            `;
            proxyList.appendChild(proxyDiv);
        }

        function generateConfig() {
            const form = document.getElementById('configForm');
            const formData = new FormData(form);

            const token = formData.get('common_token');

            const commonConfig = {
                bind_addr: formData.get('common_bind_addr'),
                bind_port: formData.get('common_bind_port'),
                token: token,
                log_level: formData.get('common_log_level'),
                log_file: formData.get('common_log_file'),
                log_max_days: formData.get('common_log_max_days')
            };

            const dashboardConfig = formData.get('dashboard_enable') ? {
                port: formData.get('dashboard_port'),
                user: formData.get('dashboard_user'),
                password: formData.get('dashboard_pwd')
            } : {};

            let clientProxyConfigs = '';
            for (let i = 1; i <= proxyCount; i++) {
                const name = formData.get(`proxy_${i}_name`);
                const type = formData.get(`proxy_${i}_type`);
                const local_ip = formData.get(`proxy_${i}_local_ip`);
                const local_port = formData.get(`proxy_${i}_local_port`);
                const remote_port = formData.get(`proxy_${i}_remote_port`);

                clientProxyConfigs += `[${name}]\ntype = ${type}\nlocal_ip = ${local_ip}\nlocal_port = ${local_port}\nremote_port = ${remote_port}\n\n`;
            }

            const serverConfigOutput = document.getElementById('serverConfigOutput');
            serverConfigOutput.textContent = `[common]\nbind_addr = ${commonConfig.bind_addr}\nbind_port = ${commonConfig.bind_port}\ntoken = ${commonConfig.token}\nlog_level = ${commonConfig.log_level}\nlog_file = ${commonConfig.log_file}\nlog_max_days = ${commonConfig.log_max_days}\n\n[dashboard]\n${Object.keys(dashboardConfig).length ? `port = ${dashboardConfig.port}\nuser = ${dashboardConfig.user}\npassword = ${dashboardConfig.password}\n` : ''}`;

            const clientConfigOutput = document.getElementById('clientConfigOutput');
            clientConfigOutput.textContent = `[common]\nserver_addr = ${formData.get('server_addr')}\nserver_port = ${formData.get('common_bind_port')}\ntoken = ${token}\n\n${clientProxyConfigs}`;
        }

        function copyConfig() {
            const serverConfigOutput = document.getElementById('serverConfigOutput').textContent;
            const clientConfigOutput = document.getElementById('clientConfigOutput').textContent;

            navigator.clipboard.writeText(`Server Config:\n${serverConfigOutput}\n\nClient Config:\n${clientConfigOutput}`)
                .then(() => alert('配置已复制到剪贴板'))
                .catch(err => alert('复制失败: ' + err));
        }
    </script>
</body>
</html>
```
