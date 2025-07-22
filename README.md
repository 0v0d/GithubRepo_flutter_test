# Flutter GitHub Repository Search チュートリアル

## 目次
1. [プロジェクトのセットアップ](#1-プロジェクトのセットアップ)
2. [依存関係の追加](#2-依存関係の追加)
3. [プロジェクト構造](#3-プロジェクト構造)
4. [モデルクラスの作成（Freezed）](#4-モデルクラスの作成freezed)
5. [ネットワーキング（Dio + Retrofit）](#5-ネットワーキングdio--retrofit)
6. [ローカルデータソース（SQLite + SharedPreferences）](#6-ローカルデータソースsqlite--sharedpreferences)
7. [リポジトリパターンの実装](#7-リポジトリパターンの実装)
8. [ナビゲーション（Auto Route）](#8-ナビゲーションauto-route)
9. [UI実装](#9-ui実装)
10. [StateManagementの実装](#10-statemanagementの実装)
11. [メイン機能の統合](#11-メイン機能の統合)

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
  sqflite: ^2.3.0
  shared_preferences: ^2.2.2
  path: ^1.8.3
  
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
│   │   │   ├── database_helper.dart
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
│       └── get_favorite_repositories.dart
├── presentation/
│   ├── pages/
│   │   ├── search/
│   │   │   ├── search_page.dart
│   │   │   └── widgets/
│   │   │       ├── repository_card.dart
│   │   │       └── search_bar.dart
│   │   ├── detail/
│   │   │   └── repository_detail_page.dart
│   │   ├── favorites/
│   │   │   └── favorites_page.dart
│   │   └── webview/
│   │       └── webview_page.dart
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

## 6. ローカルデータソース（SQLite + SharedPreferences）

### `lib/data/datasources/local/database_helper.dart`

```dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'dart:convert';
import '../../models/repository_model.dart';

class DatabaseHelper {
  static const _databaseName = "github_repos.db";
  static const _databaseVersion = 1;
  static const table = 'favorites';
  
  static const columnId = 'id';
  static const columnData = 'data';
  static const columnTimestamp = 'timestamp';

  DatabaseHelper._privateConstructor();
  static final DatabaseHelper instance = DatabaseHelper._privateConstructor();

  static Database? _database;
  
  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  Future<Database> _initDatabase() async {
    String path = join(await getDatabasesPath(), _databaseName);
    return await openDatabase(
      path,
      version: _databaseVersion,
      onCreate: _onCreate,
    );
  }
  
  Future<void> _onCreate(Database db, int version) async {
    await db.execute('''
      CREATE TABLE $table (
        $columnId INTEGER PRIMARY KEY,
        $columnData TEXT NOT NULL,
        $columnTimestamp INTEGER NOT NULL
      )
    ''');
  }
  
  Future<int> insertFavorite(RepositoryModel repository) async {
    Database db = await database;
    Map<String, dynamic> row = {
      columnId: repository.id,
      columnData: jsonEncode(repository.toJson()),
      columnTimestamp: DateTime.now().millisecondsSinceEpoch,
    };
    return await db.insert(
      table,
      row,
      conflictAlgorithm: ConflictAlgorithm.replace,
    );
  }
  
  Future<List<RepositoryModel>> getFavorites() async {
    Database db = await database;
    List<Map<String, dynamic>> maps = await db.query(
      table,
      orderBy: '$columnTimestamp DESC',
    );
    
    return List.generate(maps.length, (i) {
      return RepositoryModel.fromJson(
        jsonDecode(maps[i][columnData]) as Map<String, dynamic>,
      );
    });
  }
  
  Future<int> deleteFavorite(int id) async {
    Database db = await database;
    return await db.delete(
      table,
      where: '$columnId = ?',
      whereArgs: [id],
    );
  }
  
  Future<bool> isFavorite(int id) async {
    Database db = await database;
    List<Map<String, dynamic>> maps = await db.query(
      table,
      where: '$columnId = ?',
      whereArgs: [id],
    );
    return maps.isNotEmpty;
  }
}
```

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
  
  Future<List<RepositoryEntity>> getFavorites();
  Future<void> addToFavorites(RepositoryEntity repository);
  Future<void> removeFromFavorites(int id);
  Future<bool> isFavorite(int id);
  
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
import '../datasources/local/database_helper.dart';
import '../datasources/local/preferences_helper.dart';
import '../datasources/remote/github_api_service.dart';
import '../models/repository_model.dart';
import '../../core/constants/api_constants.dart';

class GithubRepositoryImpl implements GithubRepository {
  final GithubApiService _apiService;
  final DatabaseHelper _databaseHelper;
  final PreferencesHelper _preferencesHelper;
  
  GithubRepositoryImpl({
    required GithubApiService apiService,
    required DatabaseHelper databaseHelper,
    required PreferencesHelper preferencesHelper,
  })  : _apiService = apiService,
        _databaseHelper = databaseHelper,
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
  Future<List<RepositoryEntity>> getFavorites() async {
    final models = await _databaseHelper.getFavorites();
    return models.map((model) => _mapModelToEntity(model)).toList();
  }
  
  @override
  Future<void> addToFavorites(RepositoryEntity repository) async {
    final model = _mapEntityToModel(repository);
    await _databaseHelper.insertFavorite(model);
  }
  
  @override
  Future<void> removeFromFavorites(int id) async {
    await _databaseHelper.deleteFavorite(id);
  }
  
  @override
  Future<bool> isFavorite(int id) async {
    return await _databaseHelper.isFavorite(id);
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
  
  RepositoryModel _mapEntityToModel(RepositoryEntity entity) {
    return RepositoryModel(
      id: entity.id,
      name: entity.name,
      fullName: entity.fullName,
      owner: OwnerModel(
        id: 0, // Dummy value
        login: entity.ownerName,
        avatarUrl: entity.ownerAvatarUrl,
        htmlUrl: '', // Dummy value
      ),
      htmlUrl: entity.htmlUrl,
      description: entity.description,
      starsCount: entity.starsCount,
      watchersCount: 0, // Dummy value
      forksCount: entity.forksCount,
      language: entity.language,
      openIssuesCount: 0, // Dummy value
      createdAt: entity.createdAt.toIso8601String(),
      updatedAt: entity.updatedAt.toIso8601String(),
    );
  }
}
```

## 8. ナビゲーション（Auto Route）

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
          page: FavoritesRoute.page,
        ),
        AutoRoute(
          page: WebViewRoute.page,
        ),
      ];
}
```

## 9. UI実装

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
      ref.read(searchNotifierProvider.notifier).loadMore();
    }
  }
  
  @override
  Widget build(BuildContext context) {
    final searchState = ref.watch(searchNotifierProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('GitHub Repository Search'),
        actions: [
          IconButton(
            icon: const Icon(Icons.favorite),
            onPressed: () {
              context.router.push(const FavoritesRoute());
            },
          ),
          PopupMenuButton<String>(
            onSelected: (value) {
              ref.read(searchNotifierProvider.notifier).setSortOption(value);
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
              ref.read(searchNotifierProvider.notifier).search(query);
            },
            searchHistory: searchState.searchHistory,
            onHistoryTap: (query) {
              _searchController.text = query;
              ref.read(searchNotifierProvider.notifier).search(query);
            },
          ),
          Expanded(
            child: searchState.isLoading && searchState.repositories.isEmpty
                ? const Center(child: CircularProgressIndicator())
                : searchState.repositories.isEmpty
                    ? const Center(
                        child: Text('リポジトリを検索してください'),
                      )
                    : RefreshIndicator(
                        onRefresh: () async {
                          ref.read(searchNotifierProvider.notifier).refresh();
                        },
                        child: ListView.builder(
                          controller: _scrollController,
                          itemCount: searchState.repositories.length +
                              (searchState.isLoadingMore ? 1 : 0),
                          itemBuilder: (context, index) {
                            if (index == searchState.repositories.length) {
                              return const Center(
                                child: Padding(
                                  padding: EdgeInsets.all(16.0),
                                  child: CircularProgressIndicator(),
                                ),
                              );
                            }
                            
                            final repository = searchState.repositories[index];
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
import 'package:flutter_svg/flutter_svg.dart';
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

### `lib/presentation/pages/detail/repository_detail_page.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:auto_route/auto_route.dart';
import 'package:url_launcher/url_launcher.dart';
import 'package:intl/intl.dart';
import '../../../domain/entities/repository_entity.dart';
import '../../providers/github_providers.dart';
import '../../routes/app_router.gr.dart';

@RoutePage()
class RepositoryDetailPage extends ConsumerStatefulWidget {
  final RepositoryEntity repository;
  
  const RepositoryDetailPage({
    Key? key,
    required this.repository,
  }) : super(key: key);
  
  @override
  ConsumerState<RepositoryDetailPage> createState() =>
      _RepositoryDetailPageState();
}

class _RepositoryDetailPageState extends ConsumerState<RepositoryDetailPage> {
  bool _isFavorite = false;
  
  @override
  void initState() {
    super.initState();
    _checkFavoriteStatus();
  }
  
  Future<void> _checkFavoriteStatus() async {
    final isFavorite = await ref
        .read(githubRepositoryProvider)
        .isFavorite(widget.repository.id);
    setState(() {
      _isFavorite = isFavorite;
    });
  }
  
  Future<void> _toggleFavorite() async {
    final repository = ref.read(githubRepositoryProvider);
    if (_isFavorite) {
      await repository.removeFromFavorites(widget.repository.id);
    } else {
      await repository.addToFavorites(widget.repository);
    }
    setState(() {
      _isFavorite = !_isFavorite;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    final dateFormat = DateFormat('yyyy年MM月dd日');
    
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.repository.name),
        actions: [
          IconButton(
            icon: Icon(
              _isFavorite ? Icons.favorite : Icons.favorite_border,
              color: _isFavorite ? Colors.red : null,
            ),
            onPressed: _toggleFavorite,
          ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Owner Info
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Row(
                  children: [
                    CircleAvatar(
                      radius: 40,
                      backgroundImage: NetworkImage(widget.repository.ownerAvatarUrl),
                    ),
                    const SizedBox(width: 16),
                    Expanded(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text(
                            widget.repository.fullName,
                            style: Theme.of(context).textTheme.titleLarge,
                          ),
                          const SizedBox(height: 4),
                          Text(
                            'by ${widget.repository.ownerName}',
                            style: Theme.of(context).textTheme.bodyMedium,
                          ),
                        ],
                      ),
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            
            // Description
            if (widget.repository.description != null) ...[
              Card(
                child: Padding(
                  padding: const EdgeInsets.all(16),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        '説明',
                        style: Theme.of(context).textTheme.titleMedium,
                      ),
                      const SizedBox(height: 8),
                      Text(widget.repository.description!),
                    ],
                  ),
                ),
              ),
              const SizedBox(height: 16),
            ],
            
            // Stats
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      '統計情報',
                      style: Theme.of(context).textTheme.titleMedium,
                    ),
                    const SizedBox(height: 16),
                    _buildStatRow(
                      Icons.star,
                      'Stars',
                      widget.repository.starsCount.toString(),
                    ),
                    const Divider(),
                    _buildStatRow(
                      Icons.call_split,
                      'Forks',
                      widget.repository.forksCount.toString(),
                    ),
                    if (widget.repository.language != null) ...[
                      const Divider(),
                      _buildStatRow(
                        Icons.code,
                        '言語',
                        widget.repository.language!,
                      ),
                    ],
                    const Divider(),
                    _buildStatRow(
                      Icons.calendar_today,
                      '作成日',
                      dateFormat.format(widget.repository.createdAt),
                    ),
                    const Divider(),
                    _buildStatRow(
                      Icons.update,
                      '更新日',
                      dateFormat.format(widget.repository.updatedAt),
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 24),
            
            // Action Buttons
            SizedBox(
              width: double.infinity,
              child: ElevatedButton.icon(
                onPressed: () {
                  context.router.push(
                    WebViewRoute(
                      url: widget.repository.htmlUrl,
                      title: widget.repository.name,
                    ),
                  );
                },
                icon: const Icon(Icons.open_in_browser),
                label: const Text('ブラウザで開く'),
                style: ElevatedButton.styleFrom(
                  padding: const EdgeInsets.all(16),
                ),
              ),
            ),
            const SizedBox(height: 12),
            SizedBox(
              width: double.infinity,
              child: OutlinedButton.icon(
                onPressed: () async {
                  final url = Uri.parse(widget.repository.htmlUrl);
                  if (await canLaunchUrl(url)) {
                    await launchUrl(url, mode: LaunchMode.externalApplication);
                  }
                },
                icon: const Icon(Icons.launch),
                label: const Text('外部ブラウザで開く'),
                style: OutlinedButton.styleFrom(
                  padding: const EdgeInsets.all(16),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildStatRow(IconData icon, String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 8),
      child: Row(
        children: [
          Icon(icon, size: 20),
          const SizedBox(width: 16),
          Text(label),
          const Spacer(),
          Text(
            value,
            style: const TextStyle(fontWeight: FontWeight.bold),
          ),
        ],
      ),
    );
  }
}
```

### `lib/presentation/pages/webview/webview_page.dart`

```dart
import 'package:flutter/material.dart';
import 'package:auto_route/auto_route.dart';
import 'package:webview_flutter/webview_flutter.dart';

@RoutePage()
class WebViewPage extends StatefulWidget {
  final String url;
  final String title;
  
  const WebViewPage({
    Key? key,
    required this.url,
    required this.title,
  }) : super(key: key);
  
  @override
  State<WebViewPage> createState() => _WebViewPageState();
}

class _WebViewPageState extends State<WebViewPage> {
  late final WebViewController _controller;
  bool _isLoading = true;
  
  @override
  void initState() {
    super.initState();
    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..setNavigationDelegate(
        NavigationDelegate(
          onPageStarted: (String url) {
            setState(() {
              _isLoading = true;
            });
          },
          onPageFinished: (String url) {
            setState(() {
              _isLoading = false;
            });
          },
        ),
      )
      ..loadRequest(Uri.parse(widget.url));
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () {
              _controller.reload();
            },
          ),
        ],
      ),
      body: Stack(
        children: [
          WebViewWidget(controller: _controller),
          if (_isLoading)
            const Center(
              child: CircularProgressIndicator(),
            ),
        ],
      ),
    );
  }
}
```

### `lib/presentation/pages/favorites/favorites_page.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:auto_route/auto_route.dart';
import '../../providers/github_providers.dart';
import '../../routes/app_router.gr.dart';
import '../search/widgets/repository_card.dart';

@RoutePage()
class FavoritesPage extends ConsumerWidget {
  const FavoritesPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final favoritesAsync = ref.watch(favoritesProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('お気に入り'),
      ),
      body: favoritesAsync.when(
        data: (favorites) {
          if (favorites.isEmpty) {
            return const Center(
              child: Text('お気に入りはまだありません'),
            );
          }
          
          return ListView.builder(
            itemCount: favorites.length,
            itemBuilder: (context, index) {
              final repository = favorites[index];
              return RepositoryCard(
                repository: repository,
                onTap: () {
                  context.router.push(
                    RepositoryDetailRoute(repository: repository),
                  );
                },
              );
            },
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(
          child: Text('エラーが発生しました: $error'),
        ),
      ),
    );
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
  
  const SearchBarWidget({
    Key? key,
    required this.controller,
    required this.onSearch,
    required this.searchHistory,
    required this.onHistoryTap,
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
          ),
      ],
    );
  }
}
```

## 10. StateManagementの実装

### `lib/presentation/providers/github_providers.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:dio/dio.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:pretty_dio_logger/pretty_dio_logger.dart';
import '../../data/datasources/local/database_helper.dart';
import '../../data/datasources/local/preferences_helper.dart';
import '../../data/datasources/remote/github_api_service.dart';
import '../../data/repositories/github_repository_impl.dart';
import '../../domain/entities/repository_entity.dart';
import '../../domain/repositories/github_repository.dart';

// Dio Provider
final dioProvider = Provider<Dio>((ref) {
  final dio = Dio();
  dio.interceptors.add(PrettyDioLogger(
    requestHeader: true,
    requestBody: true,
    responseBody: true,
    responseHeader: false,
    error: true,
    compact: true,
  ));
  return dio;
});

// SharedPreferences Provider
final sharedPreferencesProvider = Provider<SharedPreferences>((ref) {
  throw UnimplementedError();
});

// Repository Provider
final githubRepositoryProvider = Provider<GithubRepository>((ref) {
  final dio = ref.watch(dioProvider);
  final prefs = ref.watch(sharedPreferencesProvider);
  
  return GithubRepositoryImpl(
    apiService: GithubApiService(dio),
    databaseHelper: DatabaseHelper.instance,
    preferencesHelper: PreferencesHelper(prefs),
  );
});

// Search State
class SearchState {
  final List<RepositoryEntity> repositories;
  final bool isLoading;
  final bool isLoadingMore;
  final String? error;
  final String currentQuery;
  final int currentPage;
  final String sortOption;
  final List<String> searchHistory;
  
  SearchState({
    this.repositories = const [],
    this.isLoading = false,
    this.isLoadingMore = false,
    this.error,
    this.currentQuery = '',
    this.currentPage = 1,
    this.sortOption = 'stars',
    this.searchHistory = const [],
  });
  
  SearchState copyWith({
    List<RepositoryEntity>? repositories,
    bool? isLoading,
    bool? isLoadingMore,
    String? error,
    String? currentQuery,
    int? currentPage,
    String? sortOption,
    List<String>? searchHistory,
  }) {
    return SearchState(
      repositories: repositories ?? this.repositories,
      isLoading: isLoading ?? this.isLoading,
      isLoadingMore: isLoadingMore ?? this.isLoadingMore,
      error: error,
      currentQuery: currentQuery ?? this.currentQuery,
      currentPage: currentPage ?? this.currentPage,
      sortOption: sortOption ?? this.sortOption,
      searchHistory: searchHistory ?? this.searchHistory,
    );
  }
}

// Search Notifier
class SearchNotifier extends StateNotifier<SearchState> {
  final GithubRepository _repository;
  
  SearchNotifier(this._repository) : super(SearchState()) {
    _init();
  }
  
  Future<void> _init() async {
    final history = await _repository.getSearchHistory();
    final sortOption = await _repository.getSortOption();
    state = state.copyWith(
      searchHistory: history,
      sortOption: sortOption,
    );
  }
  
  Future<void> search(String query) async {
    if (query.isEmpty) return;
    
    state = state.copyWith(
      isLoading: true,
      error: null,
      currentQuery: query,
      currentPage: 1,
      repositories: [],
    );
    
    try {
      final results = await _repository.searchRepositories(
        query: query,
        page: 1,
        sort: state.sortOption,
      );
      
      await _repository.addToSearchHistory(query);
      final history = await _repository.getSearchHistory();
      
      state = state.copyWith(
        repositories: results,
        isLoading: false,
        searchHistory: history,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        error: e.toString(),
      );
    }
  }
  
  Future<void> loadMore() async {
    if (state.isLoadingMore || state.currentQuery.isEmpty) return;
    
    state = state.copyWith(isLoadingMore: true);
    
    try {
      final nextPage = state.currentPage + 1;
      final results = await _repository.searchRepositories(
        query: state.currentQuery,
        page: nextPage,
        sort: state.sortOption,
      );
      
      state = state.copyWith(
        repositories: [...state.repositories, ...results],
        isLoadingMore: false,
        currentPage: nextPage,
      );
    } catch (e) {
      state = state.copyWith(
        isLoadingMore: false,
        error: e.toString(),
      );
    }
  }
  
  Future<void> refresh() async {
    if (state.currentQuery.isEmpty) return;
    await search(state.currentQuery);
  }
  
  Future<void> setSortOption(String option) async {
    await _repository.setSortOption(option);
    state = state.copyWith(sortOption: option);
    if (state.currentQuery.isNotEmpty) {
      await search(state.currentQuery);
    }
  }
}

// Search Provider
final searchNotifierProvider =
    StateNotifierProvider<SearchNotifier, SearchState>((ref) {
  return SearchNotifier(ref.watch(githubRepositoryProvider));
});

// Favorites Provider
final favoritesProvider = FutureProvider<List<RepositoryEntity>>((ref) async {
  final repository = ref.watch(githubRepositoryProvider);
  return await repository.getFavorites();
});
```

## 11. メイン機能の統合

### `lib/injection_container.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'presentation/providers/github_providers.dart';

Future<void> initializeDependencies() async {
  final sharedPreferences = await SharedPreferences.getInstance();
  
  // Override the provider with the actual instance
  final container = ProviderContainer(
    overrides: [
      sharedPreferencesProvider.overrideWithValue(sharedPreferences),
    ],
  );
}
```

### `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'injection_container.dart';
import 'presentation/routes/app_router.dart';
import 'presentation/providers/github_providers.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize SharedPreferences
  final sharedPreferences = await SharedPreferences.getInstance();
  
  runApp(
    ProviderScope(
      overrides: [
        sharedPreferencesProvider.overrideWithValue(sharedPreferences),
      ],
      child: MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  MyApp({Key? key}) : super(key: key);
  
  final _appRouter = AppRouter();
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'GitHub Repository Search',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      routerConfig: _appRouter.config(),
    );
  }
}
```

## 実行手順

1. **コード生成の実行**
```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

2. **アプリの起動**
```bash
flutter run
```

## 追加の考慮事項

### エラーハンドリング
- ネットワークエラーの適切な処理
- APIレート制限の考慮
- オフライン時の動作

### パフォーマンス最適化
- 画像のキャッシュ
- ページネーションの実装
- 検索のデバウンス処理

### セキュリティ
- APIトークンの安全な管理（必要な場合）
- HTTPSの使用

### テスト
- ユニットテスト
- ウィジェットテスト
- 統合テスト

このチュートリアルに従うことで、指定された要件を満たすGitHub Repository Searchアプリケーションを構築できます。
