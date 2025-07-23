# Flutter GitHub Repository Search チュートリアル (ViewModel版)

## 目次
1. [プロジェクトのセットアップ](#1-プロジェクトのセットアップ)
2. [依存関係の追加](#2-依存関係の追加)
3. [プロジェクト構造](#3-プロジェクト構造)
4. [モデルクラスの作成（Freezed）](#4-モデルクラスの作成freezed)
5. [ネットワーキング（Dio + Retrofit）](#5-ネットワーキングdio--retrofit)
6. [ローカルデータソース（SharedPreferences）](#6-ローカルデータソースsharedpreferences)
7. [リポジトリパターンの実装](#7-リポジトリパターンの実装)
8. [ViewModelの実装](#8-viewmodelの実装)
9. [ナビゲーション（Auto Route）](#9-ナビゲーションauto-route)
10. [UI実装](#10-ui実装)
11. [StateManagementの実装](#11-statemanagementの実装)
12. [メイン機能の統合](#12-メイン機能の統合)

## 1. プロジェクトのセットアップ

```bash
flutter create github_repository_search
cd github_repository_search
```

## 2. 依存関係の追加

`pubspec.yaml` ファイルに以下の依存関係を追加：

```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # Networking
  dio: ^5.4.0
  retrofit: ^4.0.3
  pretty_dio_logger: ^1.3.1
  
  # Local Storage
  shared_preferences: ^2.2.2
  
  # Model & Code Generation
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1
  
  # UI
  flutter_svg: ^2.0.9
  webview_flutter: ^4.4.2
  url_launcher: ^6.2.2
  
  # Navigation
  auto_route: ^7.8.4
  
  # State Management
  flutter_riverpod: ^2.4.9
  
  # Others
  intl: ^0.18.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1
  
  # Code Generation
  build_runner: ^2.4.7
  freezed: ^2.4.6
  json_serializable: ^6.7.1
  retrofit_generator: ^8.0.6
  auto_route_generator: ^7.3.2
```

## 3. プロジェクト構造

```
lib/
├── main.dart
├── core/
│   ├── constants/
│   │   └── api_constants.dart
│   ├── errors/
│   │   └── failures.dart
│   └── utils/
│       └── date_formatter.dart
├── data/
│   ├── datasources/
│   │   ├── local/
│   │   │   └── preferences_helper.dart
│   │   └── remote/
│   │       └── github_api_service.dart
│   ├── models/
│   │   ├── repository_model.dart
│   │   ├── repository_model.freezed.dart
│   │   ├── repository_model.g.dart
│   │   └── search_response_model.dart
│   └── repositories/
│       └── github_repository_impl.dart
├── domain/
│   ├── entities/
│   │   └── repository_entity.dart
│   ├── repositories/
│   │   └── github_repository.dart
│   └── usecases/
│       ├── search_repositories.dart
│       └── get_search_history.dart
├── presentation/
│   ├── pages/
│   │   ├── search/
│   │   │   ├── search_page.dart
│   │   │   └── widgets/
│   │   │       ├── repository_card.dart
│   │   │       └── search_bar.dart
│   │   ├── detail/
│   │   │   └── repository_detail_page.dart
│   │   └── webview/
│   │       └── webview_page.dart
│   ├── viewmodels/
│   │   ├── search_viewmodel.dart
│   │   └── repository_detail_viewmodel.dart
│   ├── providers/
│   │   └── github_providers.dart
│   └── routes/
│       ├── app_router.dart
│       └── app_router.gr.dart
└── injection_container.dart
```

## 4. モデルクラスの作成（Freezed）

### `lib/data/models/repository_model.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'repository_model.freezed.dart';
part 'repository_model.g.dart';

@freezed
class RepositoryModel with _$RepositoryModel {
  const factory RepositoryModel({
    required int id,
    required String name,
    @JsonKey(name: 'full_name') required String fullName,
    required OwnerModel owner,
    @JsonKey(name: 'html_url') required String htmlUrl,
    String? description,
    @JsonKey(name: 'stargazers_count') required int starsCount,
    @JsonKey(name: 'watchers_count') required int watchersCount,
    @JsonKey(name: 'forks_count') required int forksCount,
    String? language,
    @JsonKey(name: 'open_issues_count') required int openIssuesCount,
    @JsonKey(name: 'created_at') required String createdAt,
    @JsonKey(name: 'updated_at') required String updatedAt,
  }) = _RepositoryModel;

  factory RepositoryModel.fromJson(Map<String, dynamic> json) =>
      _$RepositoryModelFromJson(json);
}

@freezed
class OwnerModel with _$OwnerModel {
  const factory OwnerModel({
    required int id,
    required String login,
    @JsonKey(name: 'avatar_url') required String avatarUrl,
    @JsonKey(name: 'html_url') required String htmlUrl,
  }) = _OwnerModel;

  factory OwnerModel.fromJson(Map<String, dynamic> json) =>
      _$OwnerModelFromJson(json);
}
```

### `lib/data/models/search_response_model.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'repository_model.dart';

part 'search_response_model.freezed.dart';
part 'search_response_model.g.dart';

@freezed
class SearchResponseModel with _$SearchResponseModel {
  const factory SearchResponseModel({
    @JsonKey(name: 'total_count') required int totalCount,
    @JsonKey(name: 'incomplete_results') required bool incompleteResults,
    required List<RepositoryModel> items,
  }) = _SearchResponseModel;

  factory SearchResponseModel.fromJson(Map<String, dynamic> json) =>
      _$SearchResponseModelFromJson(json);
}
```

コード生成を実行：
```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

## 5. ネットワーキング（Dio + Retrofit）

### `lib/core/constants/api_constants.dart`

```dart
class ApiConstants {
  static const String baseUrl = 'https://api.github.com';
  static const String searchRepositories = '/search/repositories';
  static const int perPage = 30;
}
```

### `lib/data/datasources/remote/github_api_service.dart`

```dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';
import '../../../core/constants/api_constants.dart';
import '../../models/search_response_model.dart';

part 'github_api_service.g.dart';

@RestApi(baseUrl: ApiConstants.baseUrl)
abstract class GithubApiService {
  factory GithubApiService(Dio dio, {String baseUrl}) = _GithubApiService;

  @GET(ApiConstants.searchRepositories)
  Future<SearchResponseModel> searchRepositories(
    @Query('q') String query,
    @Query('page') int page,
    @Query('per_page') int perPage,
    @Query('sort') String sort,
  );
}
```

## 6. ローカルデータソース（SharedPreferences）

### `lib/data/datasources/local/preferences_helper.dart`

```dart
import 'package:shared_preferences/shared_preferences.dart';

class PreferencesHelper {
  static const String _keySearchHistory = 'search_history';
  static const String _keySortOption = 'sort_option';
  
  final SharedPreferences _prefs;
  
  PreferencesHelper(this._prefs);
  
  List<String> getSearchHistory() {
    return _prefs.getStringList(_keySearchHistory) ?? [];
  }
  
  Future<bool> addToSearchHistory(String query) async {
    List<String> history = getSearchHistory();
    history.remove(query);
    history.insert(0, query);
    if (history.length > 10) {
      history = history.sublist(0, 10);
    }
    return await _prefs.setStringList(_keySearchHistory, history);
  }
  
  Future<bool> clearSearchHistory() async {
    return await _prefs.remove(_keySearchHistory);
  }
  
  String getSortOption() {
    return _prefs.getString(_keySortOption) ?? 'stars';
  }
  
  Future<bool> setSortOption(String option) async {
    return await _prefs.setString(_keySortOption, option);
  }
}
```

## 7. リポジトリパターンの実装

### `lib/domain/entities/repository_entity.dart`

```dart
class RepositoryEntity {
  final int id;
  final String name;
  final String fullName;
  final String ownerName;
  final String ownerAvatarUrl;
  final String htmlUrl;
  final String? description;
  final int starsCount;
  final int forksCount;
  final String? language;
  final DateTime createdAt;
  final DateTime updatedAt;
  
  RepositoryEntity({
    required this.id,
    required this.name,
    required this.fullName,
    required this.ownerName,
    required this.ownerAvatarUrl,
    required this.htmlUrl,
    this.description,
    required this.starsCount,
    required this.forksCount,
    this.language,
    required this.createdAt,
    required this.updatedAt,
  });
}
```

### `lib/domain/repositories/github_repository.dart`

```dart
import '../entities/repository_entity.dart';

abstract class GithubRepository {
  Future<List<RepositoryEntity>> searchRepositories({
    required String query,
    required int page,
    required String sort,
  });
  
  Future<List<String>> getSearchHistory();
  Future<void> addToSearchHistory(String query);
  Future<void> clearSearchHistory();
  
  Future<String> getSortOption();
  Future<void> setSortOption(String option);
}
```

### `lib/data/repositories/github_repository_impl.dart`

```dart
import '../../domain/entities/repository_entity.dart';
import '../../domain/repositories/github_repository.dart';
import '../datasources/local/preferences_helper.dart';
import '../datasources/remote/github_api_service.dart';
import '../models/repository_model.dart';
import '../../core/constants/api_constants.dart';

class GithubRepositoryImpl implements GithubRepository {
  final GithubApiService _apiService;
  final PreferencesHelper _preferencesHelper;
  
  GithubRepositoryImpl({
    required GithubApiService apiService,
    required PreferencesHelper preferencesHelper,
  })  : _apiService = apiService,
        _preferencesHelper = preferencesHelper;
  
  @override
  Future<List<RepositoryEntity>> searchRepositories({
    required String query,
    required int page,
    required String sort,
  }) async {
    try {
      final response = await _apiService.searchRepositories(
        query,
        page,
        ApiConstants.perPage,
        sort,
      );
      
      return response.items.map((model) => _mapModelToEntity(model)).toList();
    } catch (e) {
      throw Exception('Failed to search repositories: $e');
    }
  }
  
  @override
  Future<List<String>> getSearchHistory() async {
    return _preferencesHelper.getSearchHistory();
  }
  
  @override
  Future<void> addToSearchHistory(String query) async {
    await _preferencesHelper.addToSearchHistory(query);
  }
  
  @override
  Future<void> clearSearchHistory() async {
    await _preferencesHelper.clearSearchHistory();
  }
  
  @override
  Future<String> getSortOption() async {
    return _preferencesHelper.getSortOption();
  }
  
  @override
  Future<void> setSortOption(String option) async {
    await _preferencesHelper.setSortOption(option);
  }
  
  RepositoryEntity _mapModelToEntity(RepositoryModel model) {
    return RepositoryEntity(
      id: model.id,
      name: model.name,
      fullName: model.fullName,
      ownerName: model.owner.login,
      ownerAvatarUrl: model.owner.avatarUrl,
      htmlUrl: model.htmlUrl,
      description: model.description,
      starsCount: model.starsCount,
      forksCount: model.forksCount,
      language: model.language,
      createdAt: DateTime.parse(model.createdAt),
      updatedAt: DateTime.parse(model.updatedAt),
    );
  }
}
```

## 8. ViewModelの実装

### `lib/presentation/viewmodels/search_viewmodel.dart`

```dart
import 'package:flutter/foundation.dart';
import '../../domain/entities/repository_entity.dart';
import '../../domain/repositories/github_repository.dart';

class SearchViewModel extends ChangeNotifier {
  final GithubRepository _repository;
  
  List<RepositoryEntity> _repositories = [];
  bool _isLoading = false;
  bool _isLoadingMore = false;
  String? _error;
  String _currentQuery = '';
  int _currentPage = 1;
  String _sortOption = 'stars';
  List<String> _searchHistory = [];
  
  SearchViewModel(this._repository) {
    _init();
  }
  
  // Getters
  List<RepositoryEntity> get repositories => _repositories;
  bool get isLoading => _isLoading;
  bool get isLoadingMore => _isLoadingMore;
  String? get error => _error;
  String get currentQuery => _currentQuery;
  String get sortOption => _sortOption;
  List<String> get searchHistory => _searchHistory;
  
  Future<void> _init() async {
    _searchHistory = await _repository.getSearchHistory();
    _sortOption = await _repository.getSortOption();
    notifyListeners();
  }
  
  Future<void> search(String query) async {
    if (query.isEmpty) return;
    
    _isLoading = true;
    _error = null;
    _currentQuery = query;
    _currentPage = 1;
    _repositories = [];
    notifyListeners();
    
    try {
      final results = await _repository.searchRepositories(
        query: query,
        page: 1,
        sort: _sortOption,
      );
      
      await _repository.addToSearchHistory(query);
      _searchHistory = await _repository.getSearchHistory();
      
      _repositories = results;
      _isLoading = false;
    } catch (e) {
      _error = e.toString();
      _isLoading = false;
    }
    
    notifyListeners();
  }
  
  Future<void> loadMore() async {
    if (_isLoadingMore || _currentQuery.isEmpty) return;
    
    _isLoadingMore = true;
    notifyListeners();
    
    try {
      final nextPage = _currentPage + 1;
      final results = await _repository.searchRepositories(
        query: _currentQuery,
        page: nextPage,
        sort: _sortOption,
      );
      
      _repositories = [..._repositories, ...results];
      _currentPage = nextPage;
      _isLoadingMore = false;
    } catch (e) {
      _error = e.toString();
      _isLoadingMore = false;
    }
    
    notifyListeners();
  }
  
  Future<void> refresh() async {
    if (_currentQuery.isEmpty) return;
    await search(_currentQuery);
  }
  
  Future<void> setSortOption(String option) async {
    await _repository.setSortOption(option);
    _sortOption = option;
    notifyListeners();
    
    if (_currentQuery.isNotEmpty) {
      await search(_currentQuery);
    }
  }
  
  Future<void> clearSearchHistory() async {
    await _repository.clearSearchHistory();
    _searchHistory = [];
    notifyListeners();
  }
}
```

### `lib/presentation/viewmodels/repository_detail_viewmodel.dart`

```dart
import 'package:flutter/foundation.dart';
import 'package:url_launcher/url_launcher.dart';
import '../../domain/entities/repository_entity.dart';

class RepositoryDetailViewModel extends ChangeNotifier {
  final RepositoryEntity repository;
  
  RepositoryDetailViewModel({required this.repository});
  
  Future<void> openInExternalBrowser() async {
    final url = Uri.parse(repository.htmlUrl);
    if (await canLaunchUrl(url)) {
      await launchUrl(url, mode: LaunchMode.externalApplication);
    }
  }
  
  String formatDate(DateTime date) {
    return '${date.year}年${date.month}月${date.day}日';
  }
}
```

## 9. ナビゲーション（Auto Route）

### `lib/presentation/routes/app_router.dart`

```dart
import 'package:auto_route/auto_route.dart';
import 'app_router.gr.dart';

@AutoRouterConfig(replaceInRouteName: 'Page,Route')
class AppRouter extends $AppRouter {
  @override
  List<AutoRoute> get routes => [
        AutoRoute(
          page: SearchRoute.page,
          initial: true,
        ),
        AutoRoute(
          page: RepositoryDetailRoute.page,
        ),
        AutoRoute(
          page: WebViewRoute.page,
        ),
      ];
}
```

## 10. UI実装

### `lib/presentation/pages/search/search_page.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:auto_route/auto_route.dart';
import '../../providers/github_providers.dart';
import '../../routes/app_router.gr.dart';
import 'widgets/repository_card.dart';
import 'widgets/search_bar.dart';

@RoutePage()
class SearchPage extends ConsumerStatefulWidget {
  const SearchPage({Key? key}) : super(key: key);

  @override
  ConsumerState<SearchPage> createState() => _SearchPageState();
}

class _SearchPageState extends ConsumerState<SearchPage> {
  final _scrollController = ScrollController();
  final _searchController = TextEditingController();
  
  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }
  
  @override
  void dispose() {
    _scrollController.dispose();
    _searchController.dispose();
    super.dispose();
  }
  
  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent * 0.8) {
      ref.read(searchViewModelProvider).loadMore();
    }
  }
  
  @override
  Widget build(BuildContext context) {
    final viewModel = ref.watch(searchViewModelProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('GitHub Repository Search'),
        actions: [
          PopupMenuButton<String>(
            onSelected: (value) {
              viewModel.setSortOption(value);
            },
            itemBuilder: (context) => [
              const PopupMenuItem(
                value: 'stars',
                child: Text('Stars'),
              ),
              const PopupMenuItem(
                value: 'forks',
                child: Text('Forks'),
              ),
              const PopupMenuItem(
                value: 'updated',
                child: Text('Recently Updated'),
              ),
            ],
          ),
        ],
      ),
      body: Column(
        children: [
          SearchBarWidget(
            controller: _searchController,
            onSearch: (query) {
              viewModel.search(query);
            },
            searchHistory: viewModel.searchHistory,
            onHistoryTap: (query) {
              _searchController.text = query;
              viewModel.search(query);
            },
            onClearHistory: () {
              viewModel.clearSearchHistory();
            },
          ),
          Expanded(
            child: viewModel.isLoading && viewModel.repositories.isEmpty
                ? const Center(child: CircularProgressIndicator())
                : viewModel.repositories.isEmpty
                    ? const Center(
                        child: Text('リポジトリを検索してください'),
                      )
                    : RefreshIndicator(
                        onRefresh: () async {
                          await viewModel.refresh();
                        },
                        child: ListView.builder(
                          controller: _scrollController,
                          itemCount: viewModel.repositories.length +
                              (viewModel.isLoadingMore ? 1 : 0),
                          itemBuilder: (context, index) {
                            if (index == viewModel.repositories.length) {
                              return const Center(
                                child: Padding(
                                  padding: EdgeInsets.all(16.0),
                                  child: CircularProgressIndicator(),
                                ),
                              );
                            }
                            
                            final repository = viewModel.repositories[index];
                            return RepositoryCard(
                              repository: repository,
                              onTap: () {
                                context.router.push(
                                  RepositoryDetailRoute(repository: repository),
                                );
                              },
                            );
                          },
                        ),
                      ),
          ),
        ],
      ),
    );
  }
}
```

### `lib/presentation/pages/search/widgets/repository_card.dart`

```dart
import 'package:flutter/material.dart';
import '../../../../domain/entities/repository_entity.dart';

class RepositoryCard extends StatelessWidget {
  final RepositoryEntity repository;
  final VoidCallback onTap;
  
  const RepositoryCard({
    Key? key,
    required this.repository,
    required this.onTap,
  }) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                children: [
                  CircleAvatar(
                    radius: 20,
                    backgroundImage: NetworkImage(repository.ownerAvatarUrl),
                  ),
                  const SizedBox(width: 12),
                  Expanded(
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text(
                          repository.fullName,
                          style: Theme.of(context).textTheme.titleMedium?.copyWith(
                                fontWeight: FontWeight.bold,
                              ),
                          maxLines: 1,
                          overflow: TextOverflow.ellipsis,
                        ),
                        Text(
                          repository.ownerName,
                          style: Theme.of(context).textTheme.bodySmall,
                        ),
                      ],
                    ),
                  ),
                ],
              ),
              if (repository.description != null) ...[
                const SizedBox(height: 12),
                Text(
                  repository.description!,
                  style: Theme.of(context).textTheme.bodyMedium,
                  maxLines: 2,
                  overflow: TextOverflow.ellipsis,
                ),
              ],
              const SizedBox(height: 12),
              Row(
                children: [
                  if (repository.language != null) ...[
                    _buildLanguageChip(repository.language!),
                    const SizedBox(width: 16),
                  ],
                  _buildStatItem(
                    context,
                    Icons.star,
                    repository.starsCount.toString(),
                  ),
                  const SizedBox(width: 16),
                  _buildStatItem(
                    context,
                    Icons.call_split,
                    repository.forksCount.toString(),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
  
  Widget _buildLanguageChip(String language) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: _getLanguageColor(language).withOpacity(0.2),
        borderRadius: BorderRadius.circular(12),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Container(
            width: 8,
            height: 8,
            decoration: BoxDecoration(
              color: _getLanguageColor(language),
              shape: BoxShape.circle,
            ),
          ),
          const SizedBox(width: 4),
          Text(
            language,
            style: const TextStyle(fontSize: 12),
          ),
        ],
      ),
    );
  }
  
  Widget _buildStatItem(BuildContext context, IconData icon, String count) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(icon, size: 16, color: Theme.of(context).hintColor),
        const SizedBox(width: 4),
        Text(
          count,
          style: Theme.of(context).textTheme.bodySmall,
        ),
      ],
    );
  }
  
  Color _getLanguageColor(String language) {
    final colors = {
      'JavaScript': Colors.yellow,
      'TypeScript': Colors.blue,
      'Python': Colors.green,
      'Java': Colors.orange,
      'Kotlin': Colors.purple,
      'Swift': Colors.orange,
      'Dart': Colors.blue,
      'Go': Colors.cyan,
      'Rust': Colors.brown,
      'Ruby': Colors.red,
      'PHP': Colors.indigo,
      'C++': Colors.pink,
      'C': Colors.grey,
      'C#': Colors.green,
    };
    return colors[language] ?? Colors.grey;
  }
}
```

### `lib/presentation/pages/search/widgets/search_bar.dart`

```dart
import 'package:flutter/material.dart';

class SearchBarWidget extends StatefulWidget {
  final TextEditingController controller;
  final Function(String) onSearch;
  final List<String> searchHistory;
  final Function(String) onHistoryTap;
  final VoidCallback onClearHistory;
  
  const SearchBarWidget({
    Key? key,
    required this.controller,
    required this.onSearch,
    required this.searchHistory,
    required this.onHistoryTap,
    required this.onClearHistory,
  }) : super(key: key);
  
  @override
  State<SearchBarWidget> createState() => _SearchBarWidgetState();
}

class _SearchBarWidgetState extends State<SearchBarWidget> {
  bool _showHistory = false;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Container(
          padding: const EdgeInsets.all(16),
          child: TextField(
            controller: widget.controller,
            decoration: InputDecoration(
              hintText: 'リポジトリを検索...',
              prefixIcon: const Icon(Icons.search),
              suffixIcon: widget.controller.text.isNotEmpty
                  ? IconButton(
                      icon: const Icon(Icons.clear),
                      onPressed: () {
                        widget.controller.clear();
                        setState(() {
                          _showHistory = false;
                        });
                      },
                    )
                  : null,
              border: OutlineInputBorder(
                borderRadius: BorderRadius.circular(8),
              ),
            ),
            onTap: () {
              setState(() {
                _showHistory = true;
              });
            },
            onChanged: (value) {
              setState(() {
                _showHistory = value.isEmpty;
              });
            },
            onSubmitted: (value) {
              if (value.isNotEmpty) {
                widget.onSearch(value);
                setState(() {
                  _showHistory = false;
                });
              }
            },
          ),
        ),
        if (_showHistory && widget.searchHistory.isNotEmpty)
          Container(
            constraints: const BoxConstraints(maxHeight: 200),
            child: Card(
              margin: const EdgeInsets.symmetric(horizontal: 16),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Flexible(
                    child: ListView.builder(
                      shrinkWrap: true,
                      itemCount: widget.searchHistory.length,
                      itemBuilder: (context, index) {
                        final query = widget.searchHistory[index];
                        return ListTile(
                          leading: const Icon(Icons.history),
                          title: Text(query),
                          onTap: () {
                            widget.onHistoryTap(query);
                            setState(() {
                              _showHistory = false;
                            });
                          },
                        );
                      },
                    ),
                  ),
                  const Divider(height: 1),
                  TextButton(
                    onPressed: () {
                      widget.onClearHistory();
                      setState(() {
                        _showHistory = false;
                      });
                    },
                    child: const Text('履歴をクリア'),
