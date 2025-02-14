# Copyright (c) 2023, Thomas Atkinson
#
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 the "License";
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

vkb__register_component(
    NAME core
    HEADERS
        include/core/util/strings.hpp
        include/core/util/error.hpp
        include/core/util/hash.hpp
    SRC
        src/strings.cpp
    LINK_LIBS
        spdlog::spdlog
)

vkb__register_tests(
    COMPONENT core
    NAME utils
    SRC
        tests/strings.test.cpp
    LINK_LIBS
        vkb__core
)