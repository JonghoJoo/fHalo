import requests
from collections import Counter
import os

# --- 설정 (본인의 정보로 수정하세요) ---
GITLAB_URL = "https://gitlab.example.com"  # GitLab 서버 주소
PRIVATE_TOKEN = "your_access_token"        # 액세스 토큰
# ---------------------------------------

headers = {"PRIVATE-TOKEN": PRIVATE_TOKEN}

def get_all_extensions():
    extension_counts = Counter()
    
    # 1. 모든 프로젝트 목록 가져오기 (페이지네이션 처리)
    project_url = f"{GITLAB_URL}/api/v4/projects"
    params = {"per_page": 100, "page": 1, "membership": True}
    
    while True:
        response = requests.get(project_url, headers=headers, params=params)
        if response.status_code != 200:
            print(f"Error fetching projects: {response.status_code}")
            break
            
        projects = response.json()
        if not projects:
            break
            
        for project in projects:
            p_id = project['id']
            p_name = project['path_with_namespace']
            print(f"Scanning: {p_name}...")
            
            # 2. 각 프로젝트의 파일 트리 재귀적(recursive) 조회
            # recursive=true: 모든 하위 폴더 포함
            # pagination: 파일이 많은 경우를 대비해 처리
            tree_url = f"{GITLAB_URL}/api/v4/projects/{p_id}/repository/tree"
            tree_params = {"recursive": True, "per_page": 100, "page": 1}
            
            while True:
                tree_res = requests.get(tree_url, headers=headers, params=tree_params)
                if tree_res.status_code != 200:
                    break
                
                items = tree_res.json()
                if not items:
                    break
                
                for item in items:
                    # 'blob'은 파일을 의미함
                    if item['type'] == 'blob':
                        _, ext = os.path.splitext(item['path'])
                        ext = ext.lower() if ext else "no_extension"
                        extension_counts[ext] += 1
                
                # 다음 페이지 확인
                if 'next' in tree_res.links:
                    tree_params['page'] += 1
                else:
                    break
        
        # 다음 프로젝트 페이지 확인
        if 'next' in response.links:
            params['page'] += 1
        else:
            break

    return extension_counts

# 실행 및 결과 출력
final_counts = get_all_extensions()

print("\n" + "="*30)
print(f"{'Extension':<15} | {'Count':<10}")
print("-" * 30)
for ext, count in final_counts.most_common():
    print(f"{ext:<15} | {count:<10}")
