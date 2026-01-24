---
title: "Konnect Observability のダッシュボードを Terraform で管理する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
---

## はじめに

Kong Konnectには、Observabilityという機能が存在し、Konnectで管理しているAPIトラフィックに対するオブザーバビリティを提供します。機能の中には、ダッシュボードが存在し、これを活用することでAPIの開発者や運用者が必要な情報をまとめておくことが可能です。こちらの記事では、そのダッシュボードをTerraformで管理するための方法について解説します。

## ダッシュボードのTerraform管理

まず、Konnect ObservabilityのダッシュボードをTerraformで管理しようと思うと、以下のようなリソース定義が必要です。

```hcl
resource "konnect_dashboard" "example_dashboard" {
  provider = konnect-beta
  name     = "Quick Summary Dashboard"
  labels = {
    "created_by" = "terraform"
  }
  definition = {
    # ...
  }
}
```

このとき、`definition` にはダッシュボードのJSON定義を格納するわけで、Konnectの画面を参照すると何やらJSONでエクスポートできそうなボタンが存在します。

![export-json-config](/images/konnect-observability-dashboard-by-terraform/export-json-config.png)
_JSONの定義をエクスポートするボタン_

例えば、ダッシュボードテンプレートに存在する `Quick Summary Dashboard` をJSONでエクスポートすると、以下のような構成になっています。

```json
{
  "tiles": [
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 6,
          "rows": 2
        },
        "position": {
          "col": 0,
          "row": 0
        }
      },
      "definition": {
        "chart": {
          "type": "timeseries_line",
          "chart_title": "Total traffic over time"
        },
        "query": {
          "metrics": ["request_count"],
          "datasource": "api_usage",
          "dimensions": ["time"]
        }
      },
      "id": "fae0f758-3f22-49ba-a79e-2aa80ae0456e"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 2,
          "rows": 2
        },
        "position": {
          "col": 0,
          "row": 2
        }
      },
      "definition": {
        "chart": {
          "type": "horizontal_bar",
          "stacked": true,
          "chart_title": "Top gateway services by requests"
        },
        "query": {
          "filters": [
            {
              "field": "gateway_service",
              "operator": "not_empty"
            }
          ],
          "metrics": ["request_count"],
          "datasource": "api_usage",
          "dimensions": ["gateway_service"]
        }
      },
      "id": "1d5742c4-3793-4cf1-af34-25843d0f1225"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 2,
          "rows": 2
        },
        "position": {
          "col": 2,
          "row": 2
        }
      },
      "definition": {
        "chart": {
          "type": "horizontal_bar",
          "stacked": true,
          "chart_title": "Top routes by requests"
        },
        "query": {
          "filters": [
            {
              "field": "route",
              "operator": "not_empty"
            }
          ],
          "metrics": ["request_count"],
          "datasource": "api_usage",
          "dimensions": ["route"]
        }
      },
      "id": "0ecad701-bb56-43fc-8484-68c609ca0404"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 2,
          "rows": 2
        },
        "position": {
          "col": 4,
          "row": 2
        }
      },
      "definition": {
        "chart": {
          "type": "horizontal_bar",
          "stacked": true,
          "chart_title": "Top consumers by requests"
        },
        "query": {
          "filters": [
            {
              "field": "consumer",
              "operator": "not_empty"
            }
          ],
          "metrics": ["request_count"],
          "datasource": "api_usage",
          "dimensions": ["consumer"]
        }
      },
      "id": "7e26322d-ed49-45db-bea2-3bcdd8100420"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 3,
          "rows": 2
        },
        "position": {
          "col": 0,
          "row": 4
        }
      },
      "definition": {
        "chart": {
          "type": "timeseries_line",
          "chart_title": "Latency breakdown over time"
        },
        "query": {
          "metrics": [
            "response_latency_p99",
            "response_latency_p95",
            "response_latency_p50"
          ],
          "datasource": "api_usage",
          "dimensions": ["time"]
        }
      },
      "id": "e40a73c3-1c52-44ca-92fc-f9047556cc71"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 3,
          "rows": 2
        },
        "position": {
          "col": 3,
          "row": 4
        }
      },
      "definition": {
        "chart": {
          "type": "timeseries_line",
          "chart_title": "Kong vs upstream latency over time"
        },
        "query": {
          "metrics": ["upstream_latency_p99", "kong_latency_p99"],
          "datasource": "api_usage",
          "dimensions": ["time"]
        }
      },
      "id": "b87e49de-f3bd-49d5-9f73-3e45e04de6b8"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 2,
          "rows": 2
        },
        "position": {
          "col": 0,
          "row": 6
        }
      },
      "definition": {
        "chart": {
          "type": "horizontal_bar",
          "stacked": true,
          "chart_title": "Slowest gateway services (average)"
        },
        "query": {
          "filters": [
            {
              "field": "gateway_service",
              "operator": "not_empty"
            }
          ],
          "metrics": ["response_latency_average"],
          "datasource": "api_usage",
          "dimensions": ["gateway_service"]
        }
      },
      "id": "a24d4315-b85d-4e5c-ab9e-27944cfaa61d"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 2,
          "rows": 2
        },
        "position": {
          "col": 2,
          "row": 6
        }
      },
      "definition": {
        "chart": {
          "type": "horizontal_bar",
          "stacked": true,
          "chart_title": "Slowest routes (average)"
        },
        "query": {
          "filters": [
            {
              "field": "route",
              "operator": "not_empty"
            }
          ],
          "metrics": ["response_latency_average"],
          "datasource": "api_usage",
          "dimensions": ["route"]
        }
      },
      "id": "6c5cd6a4-6d0b-4d22-93cf-2046087e7c67"
    },
    {
      "type": "chart",
      "layout": {
        "size": {
          "cols": 2,
          "rows": 2
        },
        "position": {
          "col": 4,
          "row": 6
        }
      },
      "definition": {
        "chart": {
          "type": "horizontal_bar",
          "stacked": true,
          "chart_title": "Slowest consumers (average)"
        },
        "query": {
          "filters": [
            {
              "field": "consumer",
              "operator": "not_empty"
            }
          ],
          "metrics": ["response_latency_average"],
          "datasource": "api_usage",
          "dimensions": ["consumer"]
        }
      },
      "id": "2b0adbdf-a344-4137-a27d-cc382b8cb21c"
    }
  ]
}
```

これをそのまま `definition` の中に入れればOK！というわけにはいきません。Terraform側のスキーマ（`definition`）を見てみましょう。

```hcl
resource "konnect_dashboard" "my_dashboard" {
  provider = konnect-beta
  definition = {
    preset_filters = [
      {
        field    = "ai_provider"
        operator = "not_in"
        value    = "{ \"see\": \"documentation\" }"
      }
    ]
    tiles = [
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = {
                chart_title = "...my_chart_title..."
                type        = "choropleth_map"
              }
              donut = {
                chart_title = "...my_chart_title..."
                type        = "donut"
              }
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = [
                  "status_code_grouped",
                ]
                filters = [
                  {
                    field    = "realm"
                    operator = "not_empty"
                    value    = "{ \"see\": \"documentation\" }"
                  }
                ]
                granularity = "twelveHourly"
                metrics = [
                  "kong_latency_p50"
                ]
                time_range = {
                  relative = {
                    time_range = "current_week"
                    type       = "relative"
                    tz         = "...my_tz..."
                  }
                }
              }
              llm_usage = {
                datasource = "llm_usage"
                dimensions = [
                  "consumer"
                ]
                filters = [
                  {
                    field    = "application"
                    operator = "empty"
                    value    = "{ \"see\": \"documentation\" }"
                  }
                ]
                granularity = "tenMinutely"
                metrics = [
                  "ai_request_count"
                ]
                time_range = {
                  absolute = {
                    end   = "2022-11-26T07:30:44.592Z"
                    start = "2022-01-09T02:25:36.303Z"
                    type  = "absolute"
                    tz    = "...my_tz..."
                  }
                }
              }
            }
          }
          layout = {
            position = {
              col = 4
              row = 5
            }
            size = {
              cols = 6
              rows = 8
            }
          }
          type = "chart"
        }
      }
    ]
  }
  labels = {
    key = "value"
  }
  name = "...my_name..."
}
```

以下のような差分があることが確認できます。

- JSON: tiles -> [ { "layout": ..., "definition": ... } ]
- Terraform: tiles -> [ { "chart": { "layout": ..., "definition": ... } } ]

ということで、ダッシュボードのTerraform化は、Terraformの`import`ブロックを使って、既存のリソースを抽出することで実現します。まずは、既存のダッシュボードのTerraform定義をimportします。

```hcl
import {
  provider = konnect-beta
  to       = konnect_dashboard.example_dashboard
  id       = "<抽出予定のダッシュボードのUUID>"
}
```

これを用いて、リソースを抽出します。

```sh
terraform plan -generate-config-out=generated.tf
```

作成された `generated.tf` は以下のようになっており、当たり前ですがきちんとTerraformから利用可能な形になっています。

```hcl
# __generated__ by Terraform
# Please review these resources and move them into your main configuration files.

# __generated__ by Terraform from "effa85a3-bfe0-4b48-aaa4-590efdfc46fe"
resource "konnect_dashboard" "example_dashboard" {
  provider = konnect-beta
  definition = {
    preset_filters = [
    ]
    tiles = [
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = null
              single_value   = null
              timeseries_line = {
                chart_title = "Total traffic over time"
                type        = "timeseries_line"
              }
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["time"]
                filters = [
                ]
                metrics = ["request_count"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 0
              row = 0
            }
            size = {
              cols = 6
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = {
                chart_title = "Top gateway services by requests"
                stacked     = true
                type        = "horizontal_bar"
              }
              single_value    = null
              timeseries_line = null
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["gateway_service"]
                filters = [
                  {
                    field    = "gateway_service"
                    operator = "not_empty"
                  },
                ]
                metrics = ["request_count"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 0
              row = 2
            }
            size = {
              cols = 2
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = {
                chart_title = "Top routes by requests"
                stacked     = true
                type        = "horizontal_bar"
              }
              single_value    = null
              timeseries_line = null
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["route"]
                filters = [
                  {
                    field    = "route"
                    operator = "not_empty"
                  },
                ]
                metrics = ["request_count"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 2
              row = 2
            }
            size = {
              cols = 2
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = {
                chart_title = "Top consumers by requests"
                stacked     = true
                type        = "horizontal_bar"
              }
              single_value    = null
              timeseries_line = null
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["consumer"]
                filters = [
                  {
                    field    = "consumer"
                    operator = "not_empty"
                  },
                ]
                metrics = ["request_count"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 4
              row = 2
            }
            size = {
              cols = 2
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = null
              single_value   = null
              timeseries_line = {
                chart_title = "Latency breakdown over time"
                type        = "timeseries_line"
              }
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["time"]
                filters = [
                ]
                metrics = ["response_latency_p99", "response_latency_p95", "response_latency_p50"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 0
              row = 4
            }
            size = {
              cols = 3
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = null
              single_value   = null
              timeseries_line = {
                chart_title = "Kong vs upstream latency over time"
                type        = "timeseries_line"
              }
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["time"]
                filters = [
                ]
                metrics = ["upstream_latency_p99", "kong_latency_p99"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 3
              row = 4
            }
            size = {
              cols = 3
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = {
                chart_title = "Slowest gateway services (average)"
                stacked     = true
                type        = "horizontal_bar"
              }
              single_value    = null
              timeseries_line = null
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["gateway_service"]
                filters = [
                  {
                    field    = "gateway_service"
                    operator = "not_empty"
                  },
                ]
                metrics = ["response_latency_average"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 0
              row = 6
            }
            size = {
              cols = 2
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = {
                chart_title = "Slowest routes (average)"
                stacked     = true
                type        = "horizontal_bar"
              }
              single_value    = null
              timeseries_line = null
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["route"]
                filters = [
                  {
                    field    = "route"
                    operator = "not_empty"
                  },
                ]
                metrics = ["response_latency_average"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 2
              row = 6
            }
            size = {
              cols = 2
              rows = 2
            }
          }
          type = "chart"
        }
      },
      {
        chart = {
          definition = {
            chart = {
              choropleth_map = null
              donut          = null
              horizontal_bar = {
                chart_title = "Slowest consumers (average)"
                stacked     = true
                type        = "horizontal_bar"
              }
              single_value    = null
              timeseries_line = null
            }
            query = {
              api_usage = {
                datasource = "api_usage"
                dimensions = ["consumer"]
                filters = [
                  {
                    field    = "consumer"
                    operator = "not_empty"
                  },
                ]
                metrics = ["response_latency_average"]
              }
              llm_usage = null
            }
          }
          layout = {
            position = {
              col = 4
              row = 6
            }
            size = {
              cols = 2
              rows = 2
            }
          }
          type = "chart"
        }
      },
    ]
  }
  name = "Quick summary dashboard"
}
```

これでGUIから作成したダッシュボードをTerraformの管理下へ置くことができるようになります。ご参考までに 🖐️
