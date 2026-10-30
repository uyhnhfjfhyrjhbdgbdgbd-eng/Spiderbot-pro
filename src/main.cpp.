#include <Geode/Geode.hpp>
#include <Geode/modify/PlayLayer.hpp>
#include <algorithm>
#include <vector>
using namespace geode::prelude;

namespace {
    struct LevelCache {
        std::vector<CCRect> hazards;
        std::vector<CCRect> solids;
        float maxW = 0.f;
    };
    LevelCache g_level;

    void sortByX(std::vector<CCRect>& v) {
        std::sort(v.begin(), v.end(), [](CCRect const& a, CCRect const& b) {
            return a.origin.x < b.origin.x;
        });
    }

    auto firstNear(std::vector<CCRect> const& v, float x) {
        float minX = x - g_level.maxW;
        return std::lower_bound(v.begin(), v.end(), minX, [](CCRect const& r, float val) {
            return r.origin.x < val;
        });
    }

    bool hits(std::vector<CCRect> const& v, CCRect const& box) {
        float endX = box.origin.x + box.size.width;
        for (auto it = firstNear(v, box.origin.x); it != v.end() && it->origin.x <= endX; ++it) {
            if (it->intersectsRect(box)) return true;
        }
        return false;
    }

    // ¿Hay algo peligroso si la araña está a altura y entre x0 y x1?
    bool danger(float y, float x0, float x1) {
        CCRect box(x0, y - 11.f, x1 - x0, 22.f);
        return hits(g_level.hazards, box) || hits(g_level.solids, box);
    }

    // Calcula a qué altura caería la araña al tocar (superficie opuesta).
    bool findDest(bool upsideDown, float x, float y, float& out) {
        float lo = x - 10.f, hi = x + 10.f;
        bool found = false;
        float best = 0.f;
        for (auto it = firstNear(g_level.solids, lo); it != g_level.solids.end() && it->origin.x <= hi; ++it) {
            if (it->origin.x + it->size.width < lo) continue;
            float bottom = it->origin.y;
            float top = it->origin.y + it->size.height;
            if (!upsideDown) {
                // gravedad normal: el teletransporte va hacia arriba
                if (bottom >= y + 14.f && (!found || bottom < best)) { best = bottom; found = true; }
            } else {
                // en el techo: el teletransporte va hacia abajo
                if (top <= y - 14.f && (!found || top > best)) { best = top; found = true; }
            }
        }
        if (!found) return false;
        out = upsideDown ? best + 15.f : best - 15.f;
        return true;
    }
}

class $modify(SpiderBotLayer, PlayLayer) {
    struct Fields {
        float lastX = 0.f;
        float cooldown = 0.f;
        bool pressed = false;
    };

    void setupHasCompleted() {
        PlayLayer::setupHasCompleted();
        g_level = LevelCache();
        for (auto obj : CCArrayExt<GameObject*>(m_objects)) {
            if (!obj) continue;
            auto type = obj->m_objectType;
            if (type != GameObjectType::Hazard && type != GameObjectType::Solid) continue;
            CCRect r = obj->getObjectRect();
            g_level.maxW = std::max(g_level.maxW, r.size.width);
            if (type == GameObjectType::Hazard) g_level.hazards.push_back(r);
            else g_level.solids.push_back(r);
        }
        sortByX(g_level.hazards);
        sortByX(g_level.solids);
    }

    void postUpdate(float dt) {
        PlayLayer::postUpdate(dt);

        auto p = m_player1;
        if (!p) return;

        // El bot nunca mantiene pulsado: suelta siempre en el frame siguiente.
        if (m_fields->pressed) {
            p->releaseButton(PlayerButton::Jump);
            m_fields->pressed = false;
        }

        if (!Mod::get()->getSettingValue<bool>("enabled")) return;
        if (p->m_isDead || !p->m_isSpider || dt <= 0.f) return;

        float x = p->getPositionX();
        float y = p->getPositionY();
        float speed = (x - m_fields->lastX) / dt;
        m_fields->lastX = x;

        if (m_fields->cooldown > 0.f) {
            m_fields->cooldown -= dt;
            return;
        }
        if (speed < 1.f) return;

        float ms = static_cast<float>(Mod::get()->getSettingValue<int64_t>("lookahead-ms"));
        float la = std::max(speed * ms / 1000.f, 25.f);
        float x0 = x - 5.f;
        float x1 = x + 15.f + la;

        if (!danger(y, x0, x1)) return;

        float destY = 0.f;
        if (!findDest(p->m_isUpsideDown, x, y, destY)) return;
        if (danger(destY, x0, x1)) return;

        p->pushButton(PlayerButton::Jump);
        m_fields->pressed = true;
        m_fields->cooldown = 0.1f;
    }
};
