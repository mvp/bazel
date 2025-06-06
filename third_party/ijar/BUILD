package(
    default_visibility = [
        "//src:__subpackages__",
        "//third_party/ijar:__subpackages__",
    ],
)

licenses(["notice"])  # Apache 2.0

cc_library(
    name = "zip",
    srcs = [
        "zip.cc",
    ] + select({
        "//src/conditions:windows": [
            "mapped_file_windows.cc",
        ],
        "//conditions:default": [
            "mapped_file_unix.cc",
        ],
    }),
    hdrs = [
        "common.h",
        "mapped_file.h",
        "zip.h",
    ],
    visibility = [
        "//src:__subpackages__",
        "//third_party/ijar:__subpackages__",
        "//tools/test:__pkg__",
    ],
    deps = [
        ":platform_utils",
        ":zlib_client",
    ] + select({
        "//src/conditions:windows": [
            "//src/main/cpp/util:errors",
            "//src/main/cpp/util:filesystem",
            "//src/main/cpp/util:logging",
            "//src/main/cpp/util:strings",
        ],
        "//conditions:default": [
        ],
    }),
)

cc_library(
    name = "zlib_client",
    srcs = ["zlib_client.cc"],
    hdrs = [
        "common.h",
        "zlib_client.h",
    ],
    deps = ["@zlib"],
)

cc_library(
    name = "platform_utils",
    srcs = ["platform_utils.cc"],
    hdrs = [
        "common.h",
        "platform_utils.h",
    ],
    visibility = ["//visibility:private"],
    deps = [
        "//src/main/cpp/util:errors",
        "//src/main/cpp/util:filesystem",
        "//src/main/cpp/util:logging",
    ],
)

cc_binary(
    name = "zipper",
    srcs = ["zip_main.cc"],
    visibility = ["//visibility:public"],
    deps = [":zip"],
)

cc_binary(
    name = "ijar",
    srcs = [
        "classfile.cc",
        "ijar.cc",
    ],
    visibility = ["//visibility:public"],
    deps = [":zip"],
)

filegroup(
    name = "srcs",
    srcs = glob(["**"]) + ["//third_party/ijar/test:srcs"],
    visibility = ["//third_party:__pkg__"],
)

filegroup(
    name = "embedded_zipper_sources",
    srcs = [
        "BUILD",
        "common.h",
        "mapped_file.h",
        "platform_utils.cc",
        "platform_utils.h",
        "zip.cc",
        "zip.h",
        "zip_main.cc",
        "zlib_client.cc",
        "zlib_client.h",
    ] + select({
        "//src/conditions:windows": [
            "mapped_file_windows.cc",
        ],
        "//conditions:default": [
            "mapped_file_unix.cc",
        ],
    }),
    visibility = ["//visibility:public"],
)

filegroup(
    name = "transitive_sources",
    srcs = [":srcs"] + ["//src/main/cpp/util:embedded_java_tools"],
    visibility = ["//visibility:public"],
)

genrule(
    name = "ijar_transitive_srcs_zip",
    srcs = [
        ":ijar_srcs_zip",
        "//src/main/cpp/util:cpp_util_with_deps_zip",
    ],
    outs = ["ijar_srcs_with_deps.zip"],
    cmd = "$(location //src:merge_zip_files) java_tools $@ $(SRCS)",
    tools = ["//src:merge_zip_files"],
    visibility = ["//visibility:public"],
)

genrule(
    name = "ijar_deploy_zip",
    srcs = [
        ":ijar",
        ":zipper",
    ],
    outs = ["ijar_deploy.zip"],
    cmd = "$(location //src:zip_files) java_tools/ijar $@ $(SRCS)",
    tools = ["//src:zip_files"],
    visibility = ["//visibility:public"],
)

genrule(
    name = "ijar_srcs_zip",
    srcs = glob(
        ["**"],
        exclude = ["BUILD"],
    ),
    outs = ["ijar_srcs.zip"],
    cmd = "$(location //src:zip_files) ijar $@ $(SRCS)",
    tools = ["//src:zip_files"],
    visibility = ["//visibility:private"],
)
