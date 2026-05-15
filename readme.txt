import requests
from collections import Counter
import os
import time

# --- 설정 ---
GITLAB_URL = "https://gitlab.example.com"  # GitLab 서버 주소
PRIVATE_TOKEN = "your_access_token"        # 개인 액세스 토큰
# ------------

headers = {"PRIVATE-TOKEN": PRIVATE_TOKEN}

def get_everything_extension_stats():
    # 1. 내가 접근 가능한 '모든' 프로젝트 목록 가져오기
    # membership=True: 내가 멤버로 속한 모든 그룹/개인 프로젝트 포함
    # simple=True: 프로젝트의 상세 정보 대신 필요한 최소 정보만 가져와 서버 부하 감소
    project_url = f"{GITLAB_URL}/api/v4/projects"
    params = {
        "per_page": 100, 
        "page": 1, 
        "membership": True,
        "simple": True,
        "archived": False  # 보관된 프로젝트는 제외 (원치 않으시면 삭제)
    }
    
    all_repo_stats = {}

    print("GitLab 프로젝트 목록을 불러오는 중...")

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
            p_name = project['path_with_namespace'] # '그룹/레포명' 형태
            
            print(f"  [진행중] {p_name} 스캔 중...", end='\r')
            extension_counts = Counter()
            
            # 2. 각 프로젝트의 모든 파일 트리 조회
            tree_url = f"{GITLAB_URL}/api/v4/projects/{p_id}/repository/tree"
            tree_params = {"recursive": True, "per_page": 100, "page": 1}
            
            while True:
                try:
                    tree_res = requests.get(tree_url, headers=headers, params=tree_params)
                    if tree_res.status_code != 200:
                        break
                    
                    items = tree_res.json()
                    if not items:
                        break
                    
                    for item in items:
                        if item['type'] == 'blob':
                            _, ext = os.path.splitext(item['path'])
                            ext = ext.lower() if ext else "(no_ext)"
                            extension_counts[ext] += 1
                    
                    if 'next' in tree_res.links:
                        tree_params['page'] += 1
                        time.sleep(0.05) # 페이지 간 짧은 휴식
                    else:
                        break
                except Exception as e:
                    print(f"\n      ! {p_name} 조회 중 오류 발생: {e}")
                    break
            
            all_repo_stats[p_name] = extension_counts
            time.sleep(0.2) # 레포지토리 간 휴식

        # 다음 프로젝트 페이지(다음 100개)로 이동
        if 'next' in response.links:
            params['page'] += 1
        else:
            break

    return all_repo_stats

# --- 실행 및 결과 출력 ---
print("전체 레포지토리 조회를 시작합니다. (서버 부하 방지 대기 포함)")
stats = get_everything_extension_stats()

print("\n\n" + "="*80)
print(f"{'Repository Full Path':<50} | {'Extension Stats'}")
print("-" * 80)

# 결과가 너무 많을 수 있으니 가나다순으로 정렬해서 출력
for repo in sorted(stats.keys()):
    counts = stats[repo]
    if not counts:
        count_str = "Empty Repository"
    else:
        # 상위 5개 확장자만 출력 (너무 길어질 수 있음)
        top_exts = [f"{ext}: {c}" for ext, c in counts.most_common(10)]
        count_str = ", ".join(top_exts)
    
    print(f"{repo:<50} | {count_str}")
