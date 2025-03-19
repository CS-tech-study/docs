# 최단 경로 알고리즘

## 다익스트라 알고리즘

- 특징
    - 그리디 + 동적 계획법
    - 현재 노드에서 최선의 경로를 반복적으로 찾으면서 계산 해둔 경로를 활용해 중복된 하위 문제 해결합니다.
- 장점
    - 빠르고 효율적입니다.
- 단점
    - 간선에 음의 가중치가 있을 경우 올바른 결과를 보장하지 않습니다.
- 알고리즘
    1. start에서 모든 경로를 최대값으로 초기화
    2. 연결된 노드에 대해 최솟값 경로를 탐색해 해당 값으로 갱신
    3. 모든 노드에 대한 값이 도출(or 목표에 도달)될 때 까지 2 반복
- 코드 (java, 우선순위 큐 사용)
    
    ```java
    class Dijkstra {
        static class Node implements Comparable<Node> {
            int vertex, cost;
            public Node(int vertex, int cost) {
                this.vertex = vertex;
                this.cost = cost;
            }
            @Override
            public int compareTo(Node other) {
                return this.cost - other.cost;
            }
        }
        
        public static int[] dijkstra(int n, List<List<Node>> graph, int start) {
            int[] dist = new int[n];
            // 최대값으로 모든 경로 초기화
            Arrays.fill(dist, Integer.MAX_VALUE);
            dist[start] = 0;
            
            // 우선순위 큐를 통해 구현
            PriorityQueue<Node> pq = new PriorityQueue<>();
            pq.offer(new Node(start, 0));
            
            while (!pq.isEmpty()) {
                Node cur = pq.poll();
                int curVertex = cur.vertex;
                int curCost = cur.cost;
                // 저장한 경로값이 현재 값보다 큰 경우에만 진행
                if (curCost > dist[curVertex]) continue;
                for (Node next : graph.get(curVertex)) {
                    if (dist[next.vertex] > dist[curVertex] + next.cost) {
                        dist[next.vertex] = dist[curVertex] + next.cost;
                        pq.offer(new Node(next.vertex, dist[next.vertex]));
                    }
                }
            }
            return dist;
        }
    }
    
    ```
    

## A* 알고리즘

- 특징
    - 다익스트라 알고리즘 + 휴리스틱 (어림짐작)
    - 다익스트라는 목표 노드로부터 점점 멀어지는 노드여도 계산을 해야하는 단점이 있지만, A*는 이런 경우 계산을 하지 않습니다.
    ex) 미리 목표 노드로 부터의 직선 거리를 계산해서 따로 휴리스틱 코스트로 가지고 있으면서 가중치로 사용합니다.
- 장점
    - 필요없는 계산을 줄일 수 있습니다.
- 단점
    - 미리 휴리스틱(직선거리에 대한 정보)이 필요하기 때문에 어떠한 정보도 없는 경우 사용할 수 없습니다.
- 코드
    
    ```java
    class AStar {
        // vertex, 시작부터 현재까지의 비용 g, 휴리스틱 h, 총 비용 f = g + h
        static class Node implements Comparable<Node> {
            int vertex, g, h, f;
            public Node(int vertex, int g, int h) {
                this.vertex = vertex;
                this.g = g;
                this.h = h;
                this.f = g + h;
            }
            @Override
            public int compareTo(Node other) {
                return this.f - other.f;
            }
        }
        
        public static int aStar(int n, List<List<Node>> graph, int start, int goal, int[] heuristic) {
            int[] gScore = new int[n];
            Arrays.fill(gScore, Integer.MAX_VALUE);
            gScore[start] = 0;
            
            PriorityQueue<Node> openSet = new PriorityQueue<>();
            openSet.offer(new Node(start, 0, heuristic[start]));
            // 계산이 완료된 경우를 표시
            boolean[] closedSet = new boolean[n];
            
            while (!openSet.isEmpty()) {
                Node current = openSet.poll();
                if (current.vertex == goal) {
                    return current.g;
                }
                
                if (closedSet[current.vertex]) continue;
                closedSet[current.vertex] = true;
                
                for (Node neighbor : graph.get(current.vertex)) {
                    // 이미 계산이 완료된 노드
                    if (closedSet[neighbor.vertex]) continue;
                    int tempG = gScore[current.vertex] + neighbor.cost;
                    if (tempG < gScore[neighbor.vertex]) {
                        gScore[neighbor.vertex] = tempG;
                        openSet.offer(new Node(neighbor.vertex, tempG, heuristic[neighbor.vertex]));
                    }
                }
            }
            return -1; // 경로가 존재하지 않을 경우
        }
    }
    
    ```
    

## 벨만 포드 알고리즘

- 특징
    - 매단계마다 모든 간선을 전부 확인하면서 모든 노드간의 최단 거리를 구합니다.
    - 음의 가중치 간선을 포함한 경우도 적용 가능합니다.
- 장점
    - 음의 가중치가 포함된 그래프에도 동작이 가능합니다.
    - 사이클 전체 가중치의 합이 음의 가중치를 가진 경우를 검증할 수 있습니다.
    = 만약 음의 가중치를 가진 사이클이 존재하면 무수히 줄어듭니다.
- 단점
    - 다익스트라에 비해 비교적 느립니다.
    - 실생활에서는 음의 가중치가 있는 경우가 적으므로 사용할 일이 흔치 않을 것으로 예상됩니다.
- 알고리즘
    1. 모든 경로를 최대값으로 초기화
    2. 아래 과정을 V-1번 반복
        1. 모든 간선을 하나씩 확인
        2. 각 간선을 거쳐 다른 노드로 가는 비용을 계산해 최단 거리를 갱신
- 코드
    
    ```java
    class Edge {
        int src, dest, weight;
        public Edge(int src, int dest, int weight) {
            this.src = src;
            this.dest = dest;
            this.weight = weight;
        }
    }
    
    public class BellmanFord {
        public static void bellmanFord(int V, List<Edge> edges, int src) {
            int[] dist = new int[V];
            Arrays.fill(dist, Integer.MAX_VALUE);
            dist[src] = 0;
            
            // V-1번 모든 간선을 갱신합니다.
            for (int i = 0; i < V - 1; i++) {
                for (Edge edge : edges) {
                    if (dist[edge.src] != Integer.MAX_VALUE && dist[edge.src] + edge.weight < dist[edge.dest]) {
                        dist[edge.dest] = dist[edge.src] + edge.weight;
                    }
                }
            }
            
            // 음의 사이클이 존재하는지 확인
            for (Edge edge : edges) {
                if (dist[edge.src] != Integer.MAX_VALUE && dist[edge.src] + edge.weight < dist[edge.dest]) {
                    System.out.println("음의 가중치 사이클이 존재");
                    return;
                }
            }
        }
    }
    
    ```
    

## 플로이드 워셜 알고리즘

- 특징
    - 간단하고 직관적입니다.
    - 3중 반복문을 사용해 중간 노드를 거쳐가는 모든 경로를 고려합니다.
    - 모든 노드 간의 거리를 구할 수 있습니다. (+ 음의 가중치)
- 장점
    - 모든 노드 간의 최단 경로를 얻을 수 있습니다.
- 단점
    - 음의 사이클이 없다는 전제하에만 사용할 수 있습니다.
- 알고리즘
    1. 모든 경로를 최대값으로 초기화 & 직접 연결된 노드는 해당 값으로 초기화
    2. 각 노드에 대해 아래 과정을 반복
        1. 해당 노드를 거쳐가는 경우를 계산해 최소값을 갱신 (1 → 3 vs 1 → 2 → 3) 
- 코드
    
    ```java
    public class FloydWarshall {
        public static void floydWarshall(int[][] dist) {
            int V = graph.length;
            
            // 모든 정점 쌍에 대해 중간 정점 k를 거치는 경우를 고려
            for (int k = 0; k < V; k++) {
                for (int i = 0; i < V; i++) {
                    for (int j = 0; j < V; j++) {
                        if (dist[i][k] != Integer.MAX_VALUE && 
                                dist[k][j] != Integer.MAX_VALUE &&
                                dist[i][k] + dist[k][j] < dist[i][j]) {
                            dist[i][j] = dist[i][k] + dist[k][j];
                        }
                    }
                }
            }
        }
    }
    
    ```
    

---

### 참고자료

https://roytravel.tistory.com/340

[https://velog.io/@1ncursio/에이스타-알고리즘에-대해-알아보자](https://velog.io/@1ncursio/%EC%97%90%EC%9D%B4%EC%8A%A4%ED%83%80-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EC%97%90-%EB%8C%80%ED%95%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90)