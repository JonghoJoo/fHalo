import gitlab
from collections import Counter
import os

# 1. GitLab 접속 정보 설정
GITLAB_URL = 'https://gitlab.example.com' # 본인의 GitLab 주소
PRIVATE_TOKEN = 'your_access_token'       # 개인 액세스 토큰 (API 권한 필요)

gl = gitlab.Gitlab(GITLAB_URL, private_token=PRIVATE_TOKEN)

def count_extensions():
    extension_counts = Counter()
    
    # 2. 모든 프로젝트 가져오기 (성능을 위해 get_all=True 사용)
    projects = gl.projects.list(get_all=True)
    print(f"Total projects found: {len(projects)}")

    for project in projects:
        try:
            # 3. Repository Tree를 재귀적으로 조회 (파일 목록만 가져옴)
            # recursive=True: 하위 디렉토리까지 모두 포함
            # iterator=True: 메모리 효율을 위해 제너레이터 형태로 수신
            items = project.repository_tree(recursive=True, all=True, iterator=True)
            
            for item in items:
                # 'blob' 타입이 실제 파일임 (tree는 디렉토리)
                if item['type'] == 'blob':
                    filepath = item['path']
                    _, ext = os.path.splitext(filepath)
                    
                    if ext:
                        extension_counts[ext.lower()] += 1
                    else:
                        extension_counts['no_extension'] += 1
            
            print(f"Processed: {project.path_with_namespace}")
            
        except Exception as e:
            print(f"Error processing {project.path_with_namespace}: {e}")

    return extension_counts

# 결과 출력
results = count_extensions()
print("\n--- Extension Count Results ---")
for ext, count in results.most_common():
    print(f"{ext}: {count}")

