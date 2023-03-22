# Rails Antipatterns

본 문서는 [Rails Antipatterns](https://www.amazon.com/Rails-AntiPatterns-Refactoring-Addison-Wesley-Professional/dp/0321604814/ref=sr_1_1?keywords=rails+antipatterns&qid=1679452517&sprefix=rails+anti%2Caps%2C353&sr=8-1) 라는 책을 보고 정리한 내용입니다.
각 antipattern들을 지속적으로 업데이트 하면서 링크를 생성해 나갈 예정입니다.

## Introduction

안티 패턴이란 습관적으로 많이 사용하는 패턴이지만 성능, 디버깅, 유지보수, 가독성 측면에서 부정적인 영향을 줄 수 있어 지양하는 패턴입니다. 이 책은 문서는 실수하기 쉬운 안티 패턴을 사례별로 설명하고 개선 방법을 가이드합니다.

## Chapter 1 Model

* AntiPattern: Voyeuristic Models
* AntiPattern: Fat Models
* AntiPattern: Spaghetti SQL
* AntiPattern: Duplicate Code Duplication

## Chapter 2 Domain Modeling

* AntiPattern: Authorization Astronaut
* AntiPattern: The Million-Model March

## Chapter 3 Views

* AntiPattern: PHPitis
* AntiPattern: Markup Mayhem

## Chapter 4 Controllers

* AntiPattern: Homemade Keys
* AntiPattern: Fat Controller
* AntiPattern: Bloated Sessions
* AntiPattern: Monolithic Controllers
* AntiPattern: Controller of Many Faces
* AntiPattern: A Lost Child Controller
* AntiPattern: Rat’s Nest Resources
* AntiPattern: Evil Twin Controllers

## Chapter 5 Services

* AntiPattern: Fire and Forget
* AntiPattern: Sluggish Services
* AntiPattern: Pitiful Page Parsing
* AntiPattern: Successful Failure
* AntiPattern: Kraken Code Base

## Chapter 6 Using Third-Party Code

* AntiPattern: Recutting the Gem
* AntiPattern: Amateur Gemologist
* AntiPattern: Vendor Junk Drawer

## Chapter 7 Testing

* AntiPattern: Fixture Blues
* AntiPattern: Lost in Isolation
* AntiPattern: Mock Suffocation
* AntiPattern: Untested Rake
* AntiPattern: Unprotected Jewels

## Chapter 8 Scaling and Deploying

* AntiPattern: Scaling Roadblocks
* AntiPattern: Disappearing Assets
* AntiPattern: Sluggish SQL
* AntiPattern: Painful Performance

## Chapter 9 Databases

* AntiPattern: Messy Migrations
* AntiPattern: Wet Validations

## Chapter 10 Building for Failure

* AntiPattern: Continual Catastrophe
* AntiPattern: Inaudible Failures


### 그 외

* [10 Rails  Antipatterns to avoid - Writing well reasoned code](https://acuments.com/10-rails-antipatterns-to-avoid-writing-well-reasoned-code.html)
* [로컬 변수를 사용하라 The Local Variable Aversion Antipattern](https://www.soulcutter.com/articles/local-variable-aversion-antipattern.html?utm_source=website&utm_medium=link&utm_campaign=acuments.com)
* [레일즈 패턴과 안티패턴에 대한 요즘 블로그 글들](https://blog.appsignal.com/series/rails-patterns-and-anti-patterns.html)
* [N+1 Query Problem과 Eager Loading](https://jinchoi.oopy.io/log/ror/5)

