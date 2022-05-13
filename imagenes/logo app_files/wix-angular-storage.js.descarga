"use strict";

angular.module("wixAngularStorage", [ "wixAngular" ]).constant("ANGULAR_STORAGE_PREFIX", "wixAngularStorage").constant("KEY_SEPARATOR", "|").constant("DEFAULT_AGE_IN_SEC", 60 * 60).constant("CLEANING_INTERVAL", 1e3 * 60 * 10).constant("CLEAN_EPSILON", 100).constant("MAX_KEY_LENGTH", 100).constant("MAX_VALUE_SIZE_IN_BYTES", 4 * 1024).constant("MAX_AGE_IN_SEC", 60 * 60 * 24 * 2).constant("MAX_STORAGE_SIZE_IN_BYTES", 1024 * 1024).constant("DATA_TYPE", "data").constant("ADHOC_TYPE", "adhoc").constant("REMOTE_TYPE", "remote").constant("wixAngularStorageErrors", {
    LOGGED_OUT: 1,
    NOT_FOUND: 2,
    RUNTIME_EXCEPTION: 3,
    SERVER_ERROR: 4,
    QUOTA_EXCEEDED: 5
});

"use strict";

(function() {
    cleanableStorage.$inject = [ "$interval", "$q", "recordUtils", "DATA_TYPE", "CLEANING_INTERVAL", "MAX_STORAGE_SIZE_IN_BYTES" ];
    function cleanableStorage($interval, $q, recordUtils, DATA_TYPE, CLEANING_INTERVAL, MAX_STORAGE_SIZE_IN_BYTES) {
        var dataKeys = [];
        var remoteAndAdhocKeys = [];
        function getValue(key) {
            return localStorage[key] && JSON.parse(localStorage[key]);
        }
        function clearRecord(key) {
            var record = getValue(key);
            if (record) {
                var recordSize = recordUtils.getRecordSize(key, record);
                delete localStorage[key];
                return recordSize;
            } else {
                return 0;
            }
        }
        function clearRecords(keys) {
            return keys.reduce(function(acc, key) {
                acc += clearRecord(key);
                return acc;
            }, 0);
        }
        function getWixCacheKeys() {
            return Object.keys(localStorage).filter(recordUtils.hasPrefix);
        }
        function getAllKeysAndValues(prefix) {
            var cacheStorage = {};
            var keys = Object.keys(localStorage).filter(function(key) {
                return key.indexOf(prefix) === 0;
            });
            keys.forEach(function(key) {
                cacheStorage[key] = getValue(key);
            });
            return cacheStorage;
        }
        function getWixCacheSize() {
            return getWixCacheKeys().reduce(function(acc, key) {
                return acc + recordUtils.getRecordSize(key, getValue(key));
            }, 0);
        }
        function loadExistingWixCacheKeys() {
            var createdAtSort = function(a, b) {
                return a.createdAt - b.createdAt;
            };
            var getKey = function(item) {
                return item.key;
            };
            dataKeys = [];
            remoteAndAdhocKeys = [];
            getWixCacheKeys().forEach(function(key) {
                var item = getValue(key);
                var arr = item.options.type === DATA_TYPE ? dataKeys : remoteAndAdhocKeys;
                arr.push({
                    key: key,
                    createdAt: item.createdAt
                });
            });
            dataKeys.sort(createdAtSort);
            remoteAndAdhocKeys.sort(createdAtSort);
            dataKeys = dataKeys.map(getKey);
            remoteAndAdhocKeys = remoteAndAdhocKeys.map(getKey);
        }
        function clearOtherUsers() {
            return clearRecords(getWixCacheKeys().filter(function(key) {
                return !recordUtils.belongsToCurrentUser(key);
            }));
        }
        function clearExpiredRecords() {
            return clearRecords(getWixCacheKeys().filter(function(cacheKey) {
                var record = getValue(cacheKey);
                return recordUtils.isExpired(record);
            }));
        }
        function clearNonExpiredRecord() {
            var arr = remoteAndAdhocKeys.length === 0 ? dataKeys : remoteAndAdhocKeys;
            var key = arr.shift();
            return clearRecord(key);
        }
        function clear(amount) {
            var requiredSpace = amount || 0;
            var clearedSpace = 0;
            clearedSpace += clearOtherUsers();
            clearedSpace += clearExpiredRecords();
            var size = getWixCacheSize();
            var removedRecordsSpace = 0;
            loadExistingWixCacheKeys();
            while (size - removedRecordsSpace > MAX_STORAGE_SIZE_IN_BYTES) {
                var removed = clearNonExpiredRecord();
                clearedSpace += removed;
                removedRecordsSpace += removed;
            }
            if (size - removedRecordsSpace < requiredSpace - clearedSpace) {
                return false;
            }
            while (clearedSpace < requiredSpace) {
                clearedSpace += clearNonExpiredRecord();
            }
            return true;
        }
        function promiseWrapper(fn) {
            var defer = $q.defer();
            try {
                var done;
                var result = fn(function() {
                    done = true;
                    defer.resolve();
                }, function() {
                    done = true;
                    defer.reject();
                });
                if (!done) {
                    defer.resolve(result);
                }
            } catch (e) {
                defer.reject();
            }
            return defer.promise;
        }
        clear();
        $interval(function() {
            clear();
        }, CLEANING_INTERVAL);
        return {
            set: function(key, value) {
                return promiseWrapper(function() {
                    localStorage[key] = JSON.stringify(value);
                });
            },
            get: function(key) {
                return promiseWrapper(function() {
                    return getValue(key);
                });
            },
            getAllWithPrefix: function(prefix) {
                return promiseWrapper(function() {
                    return getAllKeysAndValues(prefix);
                });
            },
            del: function(key) {
                return promiseWrapper(function() {
                    delete localStorage[key];
                });
            },
            clear: function(amount) {
                return promiseWrapper(function(resolve, reject) {
                    if (clear(amount)) {
                        resolve();
                    } else {
                        reject();
                    }
                });
            }
        };
    }
    angular.module("wixAngularStorage").factory("cleanableStorage", cleanableStorage);
})();

"use strict";

var WixCache = function() {
    WixCache.$inject = [ "provider", "$q", "recordUtils", "cleanableStorage", "wixAngularStorageErrors", "DEFAULT_AGE_IN_SEC", "DATA_TYPE", "ADHOC_TYPE", "REMOTE_TYPE", "CLEAN_EPSILON" ];
    function WixCache(provider, $q, recordUtils, cleanableStorage, wixAngularStorageErrors, DEFAULT_AGE_IN_SEC, DATA_TYPE, ADHOC_TYPE, REMOTE_TYPE, CLEAN_EPSILON) {
        this.$q = $q;
        this.recordUtils = recordUtils;
        this.cleanableStorage = cleanableStorage;
        this.wixAngularStorageErrors = wixAngularStorageErrors;
        this.DEFAULT_AGE_IN_SEC = DEFAULT_AGE_IN_SEC;
        this.DATA_TYPE = DATA_TYPE;
        this.ADHOC_TYPE = ADHOC_TYPE;
        this.REMOTE_TYPE = REMOTE_TYPE;
        this.CLEAN_EPSILON = CLEAN_EPSILON;
        this.namespace = provider.namespace;
    }
    WixCache.prototype.rejectUserNotLoggedIn = function() {
        return this.$q.reject(this.wixAngularStorageErrors.LOGGED_OUT);
    };
    WixCache.prototype.rejectWithRuntimeException = function() {
        return this.$q.reject(this.wixAngularStorageErrors.RUNTIME_EXCEPTION);
    };
    WixCache.prototype.tryToSet = function(key, value) {
        var _this = this;
        var cacheKey = this.recordUtils.getCacheKey(key, value.options);
        return this.cleanableStorage.set(cacheKey, value).then(function() {
            return key;
        }, function(reason) {
            if (reason === _this.wixAngularStorageErrors.RUNTIME_EXCEPTION) {
                return _this.rejectWithRuntimeException();
            }
            if (value.options.type === _this.REMOTE_TYPE) {
                return _this.$q.reject();
            } else {
                return _this.cleanableStorage.clear(_this.recordUtils.getRecordSize(cacheKey, value) + _this.CLEAN_EPSILON).then(function() {
                    return _this.cleanableStorage.set(cacheKey, value).then(function() {
                        return key;
                    }, function() {
                        return _this.rejectWithRuntimeException();
                    });
                }, function() {
                    return _this.$q.reject(_this.wixAngularStorageErrors.QUOTA_EXCEEDED);
                });
            }
        });
    };
    WixCache.prototype.withNamespace = function(opts) {
        var options = angular.extend({}, {
            namespace: this.namespace
        }, opts);
        this.recordUtils.validateNamespace(options);
        return options;
    };
    WixCache.prototype.set = function(key, data, options) {
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        options = this.withNamespace(options);
        this.recordUtils.validateKey(key);
        this.recordUtils.validateData(data);
        this.recordUtils.validateExpiration(options);
        var value = {
            createdAt: Date.now(),
            data: data,
            options: angular.extend({
                expiration: this.DEFAULT_AGE_IN_SEC,
                type: this.DATA_TYPE
            }, options)
        };
        return this.tryToSet(key, value);
    };
    WixCache.prototype.setWithGUID = function(data, opts) {
        if (opts === void 0) {
            opts = {};
        }
        var key = this.recordUtils.generateRandomKey();
        return this.set(key, data, angular.extend({
            expiration: null,
            type: this.ADHOC_TYPE
        }, opts));
    };
    WixCache.prototype.get = function(key, opts) {
        var _this = this;
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        opts = this.withNamespace(opts);
        return this.cleanableStorage.get(this.recordUtils.getCacheKey(key, opts)).then(function(record) {
            if (record && !_this.recordUtils.isExpired(record)) {
                return record.data;
            } else {
                return _this.$q.reject(_this.wixAngularStorageErrors.NOT_FOUND);
            }
        }, function() {
            return _this.rejectWithRuntimeException();
        });
    };
    WixCache.prototype.getAll = function(opts) {
        var _this = this;
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        opts = this.withNamespace(opts);
        return this.cleanableStorage.getAllWithPrefix(this.recordUtils.getCachePrefix(opts)).then(function(records) {
            var cachedRecords = _this.cachedRecords(records);
            if (Object.keys(cachedRecords).length === 0) {
                return _this.$q.reject(_this.wixAngularStorageErrors.NOT_FOUND);
            }
            return cachedRecords;
        }, function() {
            return _this.rejectWithRuntimeException();
        });
    };
    WixCache.prototype.remove = function(key, opts) {
        var _this = this;
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        opts = this.withNamespace(opts);
        return this.cleanableStorage.del(this.recordUtils.getCacheKey(key, opts)).catch(function() {
            return _this.rejectWithRuntimeException();
        });
    };
    WixCache.prototype.cachedRecords = function(records) {
        var _this = this;
        return Object.keys(records).reduce(function(cachedRecords, key) {
            if (records[key] && !_this.recordUtils.isExpired(records[key])) {
                var originKey = _this.recordUtils.getOriginKey(key);
                cachedRecords[originKey] = records[key].data;
            }
            return cachedRecords;
        }, {});
    };
    return WixCache;
}();

var WixCacheProvider = function() {
    function WixCacheProvider() {}
    WixCacheProvider.prototype.setNamespace = function(namespace) {
        this.namespace = namespace;
    };
    WixCacheProvider.prototype.$get = function($injector) {
        return $injector.instantiate(WixCache, {
            provider: this
        });
    };
    WixCacheProvider.prototype.$get.$inject = [ "$injector" ];
    return WixCacheProvider;
}();

angular.module("wixAngularAppInternal").provider("wixCache", WixCacheProvider);

"use strict";

var WixStorage = function() {
    WixStorage.$inject = [ "provider", "$q", "$http", "recordUtils", "wixCache", "wixAngularStorageErrors", "ANGULAR_STORAGE_PREFIX", "REMOTE_TYPE", "DEFAULT_AGE_IN_SEC" ];
    function WixStorage(provider, $q, $http, recordUtils, wixCache, wixAngularStorageErrors, ANGULAR_STORAGE_PREFIX, REMOTE_TYPE, DEFAULT_AGE_IN_SEC) {
        this.$q = $q;
        this.$http = $http;
        this.recordUtils = recordUtils;
        this.wixCache = wixCache;
        this.wixAngularStorageErrors = wixAngularStorageErrors;
        this.ANGULAR_STORAGE_PREFIX = ANGULAR_STORAGE_PREFIX;
        this.REMOTE_TYPE = REMOTE_TYPE;
        this.DEFAULT_AGE_IN_SEC = DEFAULT_AGE_IN_SEC;
        this.namespace = provider.namespace;
    }
    WixStorage.prototype.rejectUserNotLoggedIn = function() {
        return this.$q.reject(this.wixAngularStorageErrors.LOGGED_OUT);
    };
    WixStorage.prototype.cacheRemoteData = function(key, data, options) {
        if (!options.noCache) {
            return this.wixCache.set(key, data, angular.extend({}, options, {
                type: this.REMOTE_TYPE,
                expiration: this.DEFAULT_AGE_IN_SEC
            }));
        }
    };
    WixStorage.prototype.getUrl = function(path, options, key) {
        return [ "/_api/wix-user-preferences-webapp", path, options.namespace, options.siteId, key ].filter(angular.identity).join("/");
    };
    WixStorage.prototype.getRemote = function(key, options) {
        var _this = this;
        var path = options.siteId ? "getVolatilePrefForSite" : "getVolatilePrefForKey";
        var namespace = options.namespace;
        var url = this.getUrl(path, options, key);
        return this.$http.get(url).then(function(res) {
            if (res.data[key] === null) {
                return _this.rejectNotFound();
            }
            _this.cacheRemoteData(key, res.data[key], options);
            return res.data[key];
        }, function(err) {
            if (err.status === 404) {
                if (namespace !== _this.ANGULAR_STORAGE_PREFIX) {
                    return _this.handleNamespaceMigration(key, options);
                } else {
                    return _this.rejectNotFound();
                }
            }
            return _this.$q.reject(_this.wixAngularStorageErrors.SERVER_ERROR);
        });
    };
    WixStorage.prototype.getAllRemote = function(options) {
        var _this = this;
        var path = options.siteId ? "getVolatilePrefsForSite" : "getVolatilePrefs";
        var url = this.getUrl(path, options, undefined);
        return this.$http.get(url).then(function(res) {
            Object.keys(res.data).forEach(function(key) {
                return _this.cacheRemoteData(key, res.data[key], options);
            });
            return res.data;
        });
    };
    WixStorage.prototype.handleNamespaceMigration = function(key, options) {
        var _this = this;
        var newOptions = angular.extend({}, options, {
            namespace: this.ANGULAR_STORAGE_PREFIX,
            noCache: true
        });
        return this.getRemote(key, newOptions).then(function(data) {
            _this.cacheRemoteData(key, data, options);
            return _this.set(key, data, options).then(function() {
                return data;
            });
        }, function(error) {
            if (error === _this.wixAngularStorageErrors.NOT_FOUND) {
                _this.cacheRemoteData(key, null, options);
            }
            return _this.$q.reject(error);
        });
    };
    WixStorage.prototype.tryCache = function(key, options) {
        var _this = this;
        return this.wixCache.get(key, options).then(function(res) {
            return res || _this.rejectNotFound();
        }, function() {
            return _this.getRemote(key, options);
        });
    };
    WixStorage.prototype.tryCacheGetAll = function(options) {
        var _this = this;
        return this.wixCache.getAll(options).then(function(res) {
            return res || _this.rejectNotFound();
        }, function() {
            return _this.getAllRemote(options);
        });
    };
    WixStorage.prototype.rejectNotFound = function() {
        return this.$q.reject(this.wixAngularStorageErrors.NOT_FOUND);
    };
    WixStorage.prototype.withNamespace = function(opts) {
        var options = angular.extend({}, {
            namespace: this.namespace
        }, opts);
        this.recordUtils.validateNamespace(options);
        return options;
    };
    WixStorage.prototype.set = function(key, data, opts) {
        var _this = this;
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        var options = this.withNamespace(opts);
        this.recordUtils.validateKey(key);
        this.recordUtils.validateData(data);
        this.recordUtils.validateExpiration(options);
        var dto = {
            nameSpace: options.namespace,
            key: key,
            blob: data
        };
        if (options.siteId) {
            dto.siteId = options.siteId;
        }
        if (options.expiration) {
            dto.TTLInDays = Math.ceil(options.expiration / (60 * 60 * 24));
        }
        return this.$http.post("/_api/wix-user-preferences-webapp/set", dto).then(function() {
            _this.cacheRemoteData(key, data, options);
            return key;
        });
    };
    WixStorage.prototype.get = function(key, opts) {
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        var options = this.withNamespace(opts);
        return !options.noCache ? this.tryCache(key, options) : this.getRemote(key, options);
    };
    WixStorage.prototype.getAll = function(opts) {
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        var options = this.withNamespace(opts);
        return !options.noCache ? this.tryCacheGetAll(options) : this.getAllRemote(options);
    };
    WixStorage.prototype.remove = function(key, opts) {
        if (!this.recordUtils.isUserLoggedIn()) {
            return this.rejectUserNotLoggedIn();
        }
        return this.set(key, null, opts);
    };
    return WixStorage;
}();

var WixStorageProvider = function() {
    function WixStorageProvider() {}
    WixStorageProvider.prototype.setNamespace = function(namespace) {
        this.namespace = namespace;
    };
    WixStorageProvider.prototype.$get = function($injector) {
        return $injector.instantiate(WixStorage, {
            provider: this
        });
    };
    WixStorageProvider.prototype.$get.$inject = [ "$injector" ];
    return WixStorageProvider;
}();

angular.module("wixAngularStorage").provider("wixStorage", WixStorageProvider);

"use strict";

(function() {
    recordUtilsFactory.$inject = [ "wixCookies", "ANGULAR_STORAGE_PREFIX", "KEY_SEPARATOR", "MAX_KEY_LENGTH", "MAX_VALUE_SIZE_IN_BYTES", "MAX_AGE_IN_SEC" ];
    function recordUtilsFactory(wixCookies, ANGULAR_STORAGE_PREFIX, KEY_SEPARATOR, MAX_KEY_LENGTH, MAX_VALUE_SIZE_IN_BYTES, MAX_AGE_IN_SEC) {
        var recordUtils = {};
        function countBytes(str) {
            return encodeURI(str).match(/%..|./g).length;
        }
        function hasExpiration(options) {
            return options && !!options.expiration;
        }
        recordUtils.isUserLoggedIn = function() {
            return wixCookies.userGUID !== undefined;
        };
        recordUtils.validateKey = function(key) {
            if (typeof key !== "string" || key.length > MAX_KEY_LENGTH) {
                throw new Error("Key length should be no more than " + MAX_KEY_LENGTH + " chars");
            }
        };
        recordUtils.validateData = function(data) {
            var val = JSON.stringify(data);
            if (countBytes(val) > MAX_VALUE_SIZE_IN_BYTES) {
                throw new Error("The size of passed data exceeds the allowed " + MAX_VALUE_SIZE_IN_BYTES / 1024 + " KB");
            }
        };
        recordUtils.validateExpiration = function(options) {
            if (hasExpiration(options) && (typeof options.expiration !== "number" || options.expiration > MAX_AGE_IN_SEC)) {
                throw new Error("Expiration should be a number and cannot increase " + MAX_AGE_IN_SEC + " seconds");
            }
        };
        recordUtils.validateNamespace = function(options) {
            if (!options.namespace) {
                throw new Error("namespace is required");
            } else if (typeof options.namespace !== "string") {
                throw new Error("namespace should be a string");
            }
        };
        recordUtils.isExpired = function(record) {
            if (hasExpiration(record.options)) {
                return record.createdAt + record.options.expiration * 1e3 <= Date.now();
            } else {
                return false;
            }
        };
        recordUtils.getRecordSize = function(key, value) {
            return countBytes(key) + countBytes(JSON.stringify(value));
        };
        recordUtils.getCachePrefix = function(opts) {
            var options = opts || {};
            return [ ANGULAR_STORAGE_PREFIX, wixCookies.userGUID, options.siteId, options.namespace ].filter(angular.identity).join(KEY_SEPARATOR) + KEY_SEPARATOR;
        };
        recordUtils.getCacheKey = function(key, opts) {
            return recordUtils.getCachePrefix(opts) + key;
        };
        recordUtils.getOriginKey = function(key) {
            return key.split(KEY_SEPARATOR).pop();
        };
        recordUtils.generateRandomKey = function() {
            return Math.random().toString(36).slice(2);
        };
        recordUtils.hasPrefix = function(key) {
            return key.indexOf(ANGULAR_STORAGE_PREFIX) === 0;
        };
        recordUtils.belongsToCurrentUser = function(key) {
            if (recordUtils.isUserLoggedIn()) {
                return key.split(KEY_SEPARATOR)[1] === wixCookies.userGUID;
            } else {
                return false;
            }
        };
        return recordUtils;
    }
    angular.module("wixAngularStorage").factory("recordUtils", recordUtilsFactory);
})();