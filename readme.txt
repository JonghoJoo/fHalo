import requests
from collections import Counter
import os
import time

# --- 설정 ---
GITLAB_URL = "https://gitlab.example.com"
PRIVATE_TOKEN = "your_access_token"
GROUP_ID = "123"  # 특정 그룹 ID
# ------------

headers = {"PRIVATE-TOKEN": PRIVATE_TOKEN}

def get_repo_extension_stats():
    # 1. 특정 그룹 내의 프로젝트 목록 가져오기
    project_url = f"{GITLAB_URL}/api/v4/groups/{GROUP_ID}/projects"
    params = {"per_page": 50, "page": 1, "include_subgroups": True}
    
    all_repo_stats = {}

    while True:
        response = requests.get(project_url, headers=headers, params=params)
        if response.status_code != 200: break
        
        projects = response.json()
        if not projects: break
            
        for project in projects:
            p_id = project['id']
            p_name = project['path_with_namespace']
            
            print(f">>> Scanning Repository: {p_name}")
            extension_counts = Counter()
            
            # 2. 해당 프로젝트의 파일 트리 조회
            tree_url = f"{GITLAB_URL}/api/v4/projects/{p_id}/repository/tree"
            tree_params = {"recursive": True, "per_page": 100, "page": 1}
            
            while True:
                tree_res = requests.get(tree_url, headers=headers, params=tree_params)
                if tree_res.status_code != 200: break
                
                items = tree_res.json()
                if not items: break
                
                for item in items:
                    if item['type'] == 'blob': # 파일인 경우만
                        _, ext = os.path.splitext(item['path'])
                        ext = ext.lower() if ext else "(no_ext)"
                        extension_counts[ext] += 1
                
                if 'next' in tree_res.links:
                    tree_params['page'] += 1
                    time.sleep(0.1) # 페이지 호출 사이 짧은 휴식 (서버 보호)
                else:
                    break
            
            # 결과 저장
            all_repo_stats[p_name] = extension_counts
            time.sleep(0.3) # 프로젝트 사이 간격 (서버 보호)

        if 'next' in response.links:
            params['page'] += 1
        else:
            break

    return all_repo_stats

# --- 결과 출력 ---
stats = get_repo_extension_stats()

print("\n" + "="*50)
print(f"{'Repository Path':<40} | {'Extension Counts'}")
print("-" * 60)

for repo, counts in stats.items():
    # 각 레포별로 확장자 결과를 한 줄로 예쁘게 표현 (예: .py: 5, .md: 2)
    count_str = ", ".join([f"{ext}: {c}" for ext, c in counts.most_common()])
    print(f"{repo:<40} | {count_str}")
