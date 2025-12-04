from flask import Flask, render_template_string, redirect, url_for, session, request
import requests

# 初始化Flask应用
app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key-here'  # 生产环境请替换为随机密钥

# GitHub配置（需替换为你的OAuth应用信息）
GITHUB_CONFIG = {
    'client_id': 'your-github-client-id',
    'client_secret': 'your-github-client-secret',
    'redirect_uri': 'http://localhost:5000/github/callback',
    'api_base_url': 'https://api.github.com'
}

# 王明凯的简历数据
RESUME_DATA = {
    'personal': {
        'name': '王明凯',
        'title': '工业设计师',
        'email': '2788214191@qq.com',
        'phone': '15688495253',
        'location': '济南',
        'avatar': 'https://via.placeholder.com/200'  # 使用占位头像
    },
    'education': [
        {'school': '山东建筑大学', 'degree': '工业设计', 'period': '入学年份-毕业年份'}  # 可补充具体年份
    ],
    'experience': [],  # 暂无工作经历
    'skills': {
        'technical': ['Python', 'Rhino', 'Photoshop', 'Keyshot'],
        'soft': ['团队协作', '问题解决', '项目管理', '跨部门沟通']
    },
    'projects': [],  # 暂无项目经验
    'certificates': []  # 暂无证书荣誉
}

# GitHub API工具类
class GitHubAPI:
    def __init__(self):
        self.client_id = GITHUB_CONFIG['client_id']
        self.client_secret = GITHUB_CONFIG['client_secret']
        self.redirect_uri = GITHUB_CONFIG['redirect_uri']
        self.base_url = GITHUB_CONFIG['api_base_url']

    def get_auth_url(self):
        return (
            f"https://github.com/login/oauth/authorize"
            f"?client_id={self.client_id}&redirect_uri={self.redirect_uri}&scope=repo,user"
        )

    def get_access_token(self, code):
        url = "https://github.com/login/oauth/access_token"
        params = {
            'client_id': self.client_id,
            'client_secret': self.client_secret,
            'code': code,
            'redirect_uri': self.redirect_uri
        }
        headers = {'Accept': 'application/json'}
        response = requests.post(url, params=params, headers=headers)
        return response.json().get('access_token')

    def get_user_info(self, access_token):
        url = f"{self.base_url}/user"
        headers = {'Authorization': f'token {access_token}'}
        response = requests.get(url, headers=headers)
        return response.json()

    def get_user_repos(self, access_token):
        url = f"{self.base_url}/user/repos"
        params = {'sort': 'updated', 'per_page': 10}
        headers = {'Authorization': f'token {access_token}'}
        response = requests.get(url, params=params, headers=headers)
        return response.json()

github_api = GitHubAPI()

# 基础模板字符串
BASE_TEMPLATE = '''
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% if data %}{{ data.personal.name }}{% else %}{{ user.login }}{% endif %} - 个人简历</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css">
    <style>
        body {
            font-family: "Microsoft YaHei", sans-serif;
            background-color: #f8f9fa;
        }
        .card {
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        .badge {
            padding: 0.5rem 0.75rem;
            font-size: 0.9rem;
        }
    </style>
</head>
<body>
    <div class="container mt-5 mb-5">
        {% block content %}{% endblock %}
    </div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
'''

# 简历主页模板
INDEX_TEMPLATE = '''
{% extends "base.html" %}
{% block content %}
<div class="row mb-5">
    <div class="col-md-4 text-center">
        <img src="{{ data.personal.avatar }}" alt="头像" class="rounded-circle" width="200" height="200">
    </div>
    <div class="col-md-8 d-flex align-items-center">
        <div>
            <h1 class="display-4">{{ data.personal.name }}</h1>
            <h3 class="text-muted">{{ data.personal.title }}</h3>
            <p class="mt-3">
                <i class="bi bi-envelope"></i> {{ data.personal.email }} <br>
                <i class="bi bi-telephone"></i> {{ data.personal.phone }} <br>
                <i class="bi bi-geo-alt"></i> {{ data.personal.location }}
            </p>
            <div class="mt-4">
                <a href="{{ url_for('github_login') }}" class="btn btn-dark">
                    <i class="bi bi-github"></i> 导入GitHub数据
                </a>
            </div>
        </div>
    </div>
</div>

<div class="card mb-5">
    <div class="card-header bg-primary text-white">
        <h3><i class="bi bi-graduation-cap"></i> 教育背景</h3>
    </div>
    <div class="card-body">
        <div class="row">
            {% if data.education %}
                {% for edu in data.education %}
                <div class="col-md-6 mb-3">
                    <div class="border p-3 rounded">
                        <h5>{{ edu.school }}</h5>
                        <p class="text-muted">{{ edu.degree }} | {{ edu.period }}</p>
                    </div>
                </div>
                {% endfor %}
            {% else %}
                <div class="col-12 text-center text-muted">暂无教育背景信息</div>
            {% endif %}
        </div>
    </div>
</div>

<div class="card mb-5">
    <div class="card-header bg-primary text-white">
        <h3><i class="bi bi-briefcase"></i> 工作经历</h3>
    </div>
    <div class="card-body">
        {% if data.experience %}
            {% for exp in data.experience %}
            <div class="mb-4 border-bottom pb-3">
                <div class="d-flex justify-content-between">
                    <h5>{{ exp.company }}</h5>
                    <span class="text-muted">{{ exp.period }}</span>
                </div>
                <h6 class="text-primary">{{ exp.position }}</h6>
                <p>{{ exp.desc }}</p>
            </div>
            {% endfor %}
        {% else %}
            <div class="text-center text-muted">暂无工作经历</div>
        {% endif %}
    </div>
</div>

<div class="card mb-5">
    <div class="card-header bg-primary text-white">
        <h3><i class="bi bi-tools"></i> 技能清单</h3>
    </div>
    <div class="card-body">
        <div class="row">
            <div class="col-md-6">
                <h5>技术技能</h5>
                <div class="d-flex flex-wrap gap-2 mt-2">
                    {% for skill in data.skills.technical %}
                    <span class="badge bg-secondary">{{ skill }}</span>
                    {% endfor %}
                </div>
            </div>
            <div class="col-md-6">
                <h5>软技能</h5>
                <div class="d-flex flex-wrap gap-2 mt-2">
                    {% for skill in data.skills.soft %}
                    <span class="badge bg-info">{{ skill }}</span>
                    {% endfor %}
                </div>
            </div>
        </div>
    </div>
</div>

<div class="card mb-5">
    <div class="card-header bg-primary text-white">
        <h3><i class="bi bi-diagram-project"></i> 项目经验</h3>
    </div>
    <div class="card-body">
        {% if data.projects %}
            {% for proj in data.projects %}
            <div class="mb-3">
                <h5>{{ proj.name }} <small class="text-muted">({{ proj.period }})</small></h5>
                <p><strong>技术栈：</strong>{{ proj.tech }}</p>
                <p>{{ proj.desc }}</p>
            </div>
            {% endfor %}
        {% else %}
            <div class="text-center text-muted">暂无项目经验</div>
        {% endif %}
    </div>
</div>

<div class="card">
    <div class="card-header bg-primary text-white">
        <h3><i class="bi bi-award"></i> 证书与荣誉</h3>
    </div>
    <div class="card-body">
        {% if data.certificates %}
            <ul class="list-group list-group-flush">
                {% for cert in data.certificates %}
                <li class="list-group-item">{{ cert }}</li>
                {% endfor %}
            </ul>
        {% else %}
            <div class="text-center text-muted">暂无证书与荣誉</div>
        {% endif %}
    </div>
</div>
{% endblock %}
'''

# GitHub页面模板
GITHUB_TEMPLATE = '''
{% extends "base.html" %}
{% block content %}
<div class="row mb-5">
    <div class="col-md-3 text-center">
        <img src="{{ user.avatar_url }}" alt="GitHub头像" class="rounded-circle" width="150" height="150">
    </div>
    <div class="col-md-9 d-flex align-items-center">
        <div>
            <h1 class="display-5">
                <a href="{{ user.html_url }}" target="_blank" class="text-dark">{{ user.login }}</a>
            </h1>
            <p class="text-muted">{{ user.bio }}</p>
            <p>
                <i class="bi bi-geo-alt"></i> {{ user.location or '未知' }} | 
                <i class="bi bi-link"></i> <a href="{{ user.blog }}" target="_blank">{{ user.blog or '无' }}</a> | 
                <i class="bi bi-repo"></i> {{ user.public_repos }} 个公开仓库
            </p>
        </div>
    </div>
</div>

<div class="card mb-5">
    <div class="card-header bg-dark text-white">
        <h3><i class="bi bi-github"></i> 最新仓库</h3>
    </div>
    <div class="card-body">
        <div class="row">
            {% if repos %}
                {% for repo in repos %}
                <div class="col-md-6 mb-4">
                    <div class="border rounded p-3 h-100">
                        <h5>
                            <a href="{{ repo.html_url }}" target="_blank" class="text-primary">{{ repo.name }}</a>
                        </h5>
                        <p class="text-muted">{{ repo.description or '无描述' }}</p>
                        <div class="d-flex justify-content-between align-items-center">
                            <span class="badge bg-secondary">{{ repo.language or '未知语言' }}</span>
                            <div>
                                <span class="me-3"><i class="bi bi-star"></i> {{ repo.stargazers_count }}</span>
                                <span><i class="bi bi-code-fork"></i> {{ repo.forks_count }}</span>
                            </div>
                        </div>
                        <small class="text-muted">更新于：{{ repo.updated_at.split('T')[0] }}</small>
                    </div>
                </div>
                {% endfor %}
            {% else %}
                <div class="col-12 text-center text-muted">暂无公开仓库</div>
            {% endif %}
        </div>
    </div>
</div>

<div class="text-center">
    <a href="{{ url_for('index') }}" class="btn btn-primary">返回简历主页</a>
</div>
{% endblock %}
'''

# 路由定义
@app.route('/')
def index():
    # 注册基础模板
    app.jinja_env.from_string(BASE_TEMPLATE).name = 'base.html'
    return render_template_string(INDEX_TEMPLATE, data=RESUME_DATA)

@app.route('/github/login')
def github_login():
    auth_url = github_api.get_auth_url()
    return redirect(auth_url)

@app.route('/github/callback')
def github_callback():
    code = request.args.get('code')
    if not code:
        return redirect(url_for('index'))
    
    access_token = github_api.get_access_token(code)
    if not access_token:
        return "GitHub授权失败！"
    
    session['github_access_token'] = access_token
    user_info = github_api.get_user_info(access_token)
    repos = github_api.get_user_repos(access_token)
    
    # 注册基础模板
    app.jinja_env.from_string(BASE_TEMPLATE).name = 'base.html'
    return render_template_string(GITHUB_TEMPLATE, user=user_info, repos=repos)

@app.route('/github')
def github_page():
    access_token = session.get('github_access_token')
    if not access_token:
        return redirect(url_for('github_login'))
    
    user_info = github_api.get_user_info(access_token)
    repos = github_api.get_user_repos(access_token)
    
    # 注册基础模板
    app.jinja_env.from_string(BASE_TEMPLATE).name = 'base.html'
    return render_template_string(GITHUB_TEMPLATE, user=user_info, repos=repos)

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
