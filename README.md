# Chirpy Starter

[![Gem Version](https://img.shields.io/gem/v/jekyll-theme-chirpy)][gem]&nbsp;
[![GitHub license](https://img.shields.io/github/license/cotes2020/chirpy-starter.svg?color=blue)][mit]

When installing the [**Chirpy**][chirpy] theme through [RubyGems.org][gem], Jekyll can only read files in the folders
`_data`, `_layouts`, `_includes`, `_sass` and `assets`, as well as a small part of options of the `_config.yml` file
from the theme's gem. If you have ever installed this theme gem, you can use the command
`bundle info --path jekyll-theme-chirpy` to locate these files.

The Jekyll team claims that this is to leave the ball in the user’s court, but this also results in users not being
able to enjoy the out-of-the-box experience when using feature-rich themes.

To fully use all the features of **Chirpy**, you need to copy the other critical files from the theme's gem to your
Jekyll site. The following is a list of targets:

```shell
.
├── _config.yml
├── _plugins
├── _tabs
└── index.html
```

To save you time, and also in case you lose some files while copying, we extract those files/configurations of the
latest version of the **Chirpy** theme and the [CD][CD] workflow to here, so that you can start writing in minutes.

## Usage

Check out the [theme's docs](https://github.com/cotes2020/jekyll-theme-chirpy/wiki).

## Contributing

This repository is automatically updated with new releases from the theme repository. If you encounter any issues or want to contribute to its improvement, please visit the [theme repository][chirpy] to provide feedback.

## License

This work is published under [MIT][mit] License.

[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[CD]: https://en.wikipedia.org/wiki/Continuous_deployment
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE

## Guides
### 도커 환경 실행
vscode에서 ```ctrl+shift+p``` -> ```Dev Containers: Open Folder in Container...```
### jekyll-compose 설정
1. Gemfile에 ```gem 'jekyll-compose', group: [:jekyll_plugins]``` 추가
2. ```bundle``` 명령어 실행
### 새로운 포스트 생성 및 카테고리 설정
1. ```bundle exec jekyll compose "Title"```로 새 포스트 생성
2. _posts 폴더에 생성된 파일에 포스트 내용물 작성
    > category: [TOP_CATEGORY] // TOP_CATEGORY 대분류 안에 속함  
    > category: [TOP_CATEGORY, SUB_CATEGORY] // AAA 대분류, BBB 소분류 안에 속함
    > tags: [tag1, tag2] // tag는 소문자만 가능하다고 함
### 로컬 서버 실행해보기
1. ```bundle exec jekyll serve``` 명령어 실행
2. http://127.0.0.1:4000/ 접속
