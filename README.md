// ============================================================================
// TRAIL MAP 3D SEARCH - FIRST VERSION
// ============================================================================
//
// COORDINATE CONVENTION
//
// A = original 3D vegetation/pixel point (worldPos)
// B = vertical projection of A onto trail-map plane Z = 0
// C = first impulse center reconstructed by sampling trail map at B
//
// TrailMap:
//      R,G = normalized direction from sampled pixel -> impulse center
//      B   = normalized distance from sampled pixel -> impulse center
//      A   = impulse power / temporal decay
//
// FIRST IDEA:
//
//      A
//      |
//      |
//      B ---------------- Z = 0
//
// Sample B:
//
//      B -> C
//
// Calculate:
//
//      CA = A - C
//      distanceCA = length(CA)
//
//      CB = normalize(B - C) on the Z=0 plane
//
// First test point:
//
//      B2 = C + CB * distanceCA
//
// Sample B2:
//
//      B2 -> C
//          C is confirmed.
//          A is inside C.
//
//      B2 -> EMPTY
//          C is OUT using this first approximation.
//
//      B2 -> D1
//          D1 may simply be overwriting C.
//          Therefore check opposite side:
//
//          B3 = C - CB * distanceCA
//
//          B3 -> C
//              C confirmed.
//
//          B3 -> EMPTY
//              C out.
//              D1 is queued and tested.
//
//          B3 -> D1
//              C out.
//              D1 queued.
//
//          B3 -> D2
//              C out.
//              D1 and D2 queued.
//
// D1 / D2 use THE EXACT SAME TEST.
//
// Example:
//
//      candidate = D1
//      sourcePoint = B2
//
//      DA = A - D1
//      D1B2 = normalize(B2 - D1)
//      distanceDA = length(DA)
//
//      test1 = D1 + D1B2 * distanceDA
//      test2 = D1 - D1B2 * distanceDA
//
// ============================================================================


#define TRAIL_MAX_SAMPLES       5
#define TRAIL_MAX_CANDIDATES    5

#define TRAIL_RESULT_NONE       0
#define TRAIL_RESULT_FOUND      1
#define TRAIL_RESULT_BUDGET_END 2


static const float TRAIL_MIN_POWER       = 0.03f;
static const float TRAIL_MAX_DISTANCE    = 3.0f;

// Because RG/B may be 8-bit and reconstructed centers will not be exact.
static const float TRAIL_CENTER_EPSILON  = 0.05f;


// ============================================================================
// RESOURCES
// ============================================================================

Texture2D<float4> gTrailMap;
SamplerState      gTrailSampler;


// ============================================================================
// EXAMPLE TRAIL MAP WORLD -> UV DATA
//
// Replace this with your existing GetUVs() implementation if you already
// have one.
//
// XY is the trail-map plane.
// Z is height above/below the trail-map plane.
// ============================================================================

cbuffer TrailMapCB : register(b0)
{
    float2 gTrailWorldMin;
    float2 gTrailWorldSize;
};


// Replace this with YOUR existing GetUVs if necessary.
float2 GetUVs(float2 worldXY)
{
    return
        (worldXY - gTrailWorldMin) /
        gTrailWorldSize;
}


// ============================================================================
// DATA TYPES
// ============================================================================

struct TrailHit
{
    uint valid;

    // World-space position on Z = 0 at which we sampled.
    float3 samplePoint;

    // Decoded RG.
    // Direction:
    //
    //      samplePoint -> impulseCenter
    //
    float2 directionToCenter;

    // Decoded Blue.
    float distanceToCenter;

    // Alpha.
    float power;

    // Reconstructed center on Z = 0.
    float3 center;
};


struct TrailCandidate
{
    uint valid;

    // C / D1 / D2 / ...
    float3 center;

    // The world-space Z=0 sample which discovered this candidate.
    //
    // C:
    //      discoveredFrom = B
    //
    // D1:
    //      discoveredFrom = B2
    //
    // D2:
    //      discoveredFrom = B3
    //
    float3 discoveredFrom;

    float power;
};


struct CandidateTestResult
{
    // 1 = this candidate owns A according to our test.
    uint inside;

    // Candidate discovered on first side.
    TrailCandidate D1;

    // Candidate discovered on opposite side.
    TrailCandidate D2;

    // Debug positions.
    float3 probePositive;
    float3 probeNegative;
};


struct TrailSearchResult
{
    uint state;

    uint samplesUsed;

    float3 center;

    float power;
};


// ============================================================================
// DECODE TRAIL MAP
// ============================================================================

float2 DecodeTrailDirection(float2 packedRG)
{
    // [0,1] -> [-1,1]
    float2 direction =
        packedRG * 2.0f - 1.0f;

    float directionLengthSq =
        dot(direction, direction);

    if (directionLengthSq <= 1e-8f)
    {
        return float2(1.0f, 0.0f);
    }

    return
        direction *
        rsqrt(directionLengthSq);
}


float DecodeTrailDistance(float packedBlue)
{
    return
        packedBlue *
        TRAIL_MAX_DISTANCE;
}


// ============================================================================
// SAMPLE TRAIL MAP USING A WORLD-SPACE Z=0 POSITION
// ============================================================================

TrailHit SampleTrailMapWorld(float3 worldPoint)
{
    TrailHit hit =
        (TrailHit)0;


    // Make absolutely sure this lookup is on the trail-map plane.
    worldPoint.z = 0.0f;

    hit.samplePoint =
        worldPoint;


    // --------------------------------------------------------
    // WORLD XY -> TRAIL MAP UV
    // --------------------------------------------------------

    float2 uv =
        GetUVs(worldPoint.xy);


    if (
        any(uv < 0.0f) ||
        any(uv > 1.0f)
    )
    {
        return hit;
    }


    // --------------------------------------------------------
    // SAMPLE
    // --------------------------------------------------------

    float4 trail =
        gTrailMap.SampleLevel(
            gTrailSampler,
            uv,
            0.0f
        );


    // --------------------------------------------------------
    // ALPHA = POWER / DECAY
    //
    // Dead impulse = empty for this first implementation.
    // --------------------------------------------------------

    if (trail.a <= TRAIL_MIN_POWER)
    {
        return hit;
    }


    hit.valid =
        1;


    // --------------------------------------------------------
    // RG
    // --------------------------------------------------------

    hit.directionToCenter =
        DecodeTrailDirection(
            trail.rg
        );


    // --------------------------------------------------------
    // BLUE
    // --------------------------------------------------------

    hit.distanceToCenter =
        DecodeTrailDistance(
            trail.b
        );


    // --------------------------------------------------------
    // ALPHA
    // --------------------------------------------------------

    hit.power =
        trail.a;


    // --------------------------------------------------------
    // RECONSTRUCT CENTER
    //
    // center =
    //      samplePosition +
    //      directionToCenter *
    //      distanceToCenter
    // --------------------------------------------------------

    hit.center =
        float3(
            worldPoint.xy +
            hit.directionToCenter *
            hit.distanceToCenter,

            0.0f
        );


    return hit;
}


// ============================================================================
// CENTER COMPARISON
// ============================================================================

bool SameTrailCenter(
    float3 centerA,
    float3 centerB)
{
    float2 delta =
        centerA.xy -
        centerB.xy;


    return
        dot(delta, delta) <=
        TRAIL_CENTER_EPSILON *
        TRAIL_CENTER_EPSILON;
}


// ============================================================================
// MAKE CANDIDATE FROM TRAIL MAP HIT
// ============================================================================

TrailCandidate MakeTrailCandidate(
    TrailHit hit)
{
    TrailCandidate candidate =
        (TrailCandidate)0;


    if (!hit.valid)
    {
        return candidate;
    }


    candidate.valid =
        1;


    candidate.center =
        hit.center;


    // This is important.
    //
    // The candidate remembers WHERE it was discovered.
    //
    // C:
    //      discoveredFrom = B
    //
    // D1:
    //      discoveredFrom = B2
    //
    // This gives us the line direction when we test the candidate.
    candidate.discoveredFrom =
        hit.samplePoint;


    candidate.power =
        hit.power;


    return candidate;
}


// ============================================================================
// TEST ONE CANDIDATE
//
// For the first candidate:
//
//      A             = original worldPos
//      candidate     = C
//      sourcePoint   = B
//
// For D1:
//
//      A             = original worldPos
//      candidate     = D1
//      sourcePoint   = B2
//
// Same algorithm in both cases.
// ============================================================================

CandidateTestResult TestCandidate(
    float3 A,
    TrailCandidate candidate,
    inout uint samplesUsed)
{
    CandidateTestResult result =
        (CandidateTestResult)0;


    if (!candidate.valid)
    {
        return result;
    }


    // ========================================================================
    // CURRENT CENTER
    //
    // First iteration:
    //
    //      C = candidate.center
    //
    // Later:
    //
    //      D = candidate.center
    // ========================================================================

    float3 C =
        candidate.center;


    // ========================================================================
    // POINT WHICH DISCOVERED C
    //
    // First iteration:
    //
    //      B = candidate.discoveredFrom
    //
    // For D1:
    //
    //      B = B2
    //
    // For D2:
    //
    //      B = B3
    // ========================================================================

    float3 B =
        candidate.discoveredFrom;


    // Force both to plane.
    C.z = 0.0f;
    B.z = 0.0f;


    // ========================================================================
    // CA
    //
    // Full 3D vector from candidate center C to original point A.
    // ========================================================================

    float3 CA =
        A - C;


    // Full 3D distance C -> A.
    float distanceCA =
        length(CA);


    // ========================================================================
    // CB
    //
    // Direction from C toward B on Z = 0.
    // ========================================================================

    float2 CB =
        B.xy -
        C.xy;


    float cbLengthSq =
        dot(CB, CB);


    float2 directionCB;


    if (cbLengthSq > 1e-8f)
    {
        directionCB =
            CB *
            rsqrt(cbLengthSq);
    }
    else
    {
        // C and B are exactly the same point.
        //
        // Because the footprint is circular, any direction
        // on the plane is valid.
        directionCB =
            float2(
                1.0f,
                0.0f
            );
    }


    // ========================================================================
    // B2
    //
    // First test point.
    //
    // Start at C.
    // Travel toward B.
    // Travel exactly the 3D distance C -> A.
    //
    // Therefore:
    //
    //      distance(C, B2) == distance(C, A)
    //
    // ========================================================================

    float3 B2 =
        float3(
            C.xy +
            directionCB *
            distanceCA,

            0.0f
        );


    result.probePositive =
        B2;


    // ========================================================================
    // SAMPLE B2
    // ========================================================================

    if (
        samplesUsed >=
        TRAIL_MAX_SAMPLES
    )
    {
        return result;
    }


    TrailHit hitB2 =
        SampleTrailMapWorld(
            B2
        );


    samplesUsed++;


    // ========================================================================
    // CASE 1
    //
    // B2 gives C.
    //
    // Because:
    //
    //      distance(C, B2) == distance(C, A)
    //
    // and C owns B2:
    //
    //      realRadius(C) >= distance(C, A)
    //
    // therefore A is inside C.
    // ========================================================================

    if (
        hitB2.valid &&
        SameTrailCenter(
            hitB2.center,
            C
        )
    )
    {
        result.inside =
            1;

        return result;
    }


    // ========================================================================
    // CASE 2
    //
    // B2 is EMPTY.
    //
    // YOUR FIRST IMPLEMENTATION RULE:
    //
    //      C is OUT.
    //
    // We do NOT test the opposite side.
    //
    // This is intentionally the simple first approximation.
    // ========================================================================

    if (!hitB2.valid)
    {
        result.inside =
            0;

        return result;
    }


    // ========================================================================
    // CASE 3
    //
    // B2 contains something, but it is NOT C.
    //
    // Call it D1.
    //
    // D1 may be overwriting C at B2.
    //
    // Therefore we cannot reject C yet.
    //
    // We now check the opposite side.
    // ========================================================================

    TrailCandidate D1 =
        MakeTrailCandidate(
            hitB2
        );


    result.D1 =
        D1;


    // ========================================================================
    // B3
    //
    // Opposite direction from C.
    //
    //      B2 = C + directionCB * distanceCA
    //      B3 = C - directionCB * distanceCA
    //
    // ========================================================================

    float3 B3 =
        float3(
            C.xy -
            directionCB *
            distanceCA,

            0.0f
        );


    result.probeNegative =
        B3;


    // ========================================================================
    // SAMPLE B3
    // ========================================================================

    if (
        samplesUsed >=
        TRAIL_MAX_SAMPLES
    )
    {
        return result;
    }


    TrailHit hitB3 =
        SampleTrailMapWorld(
            B3
        );


    samplesUsed++;


    // ========================================================================
    // CASE 3A
    //
    // Opposite side gives C.
    //
    // C IS CONFIRMED.
    // ========================================================================

    if (
        hitB3.valid &&
        SameTrailCenter(
            hitB3.center,
            C
        )
    )
    {
        result.inside =
            1;

        return result;
    }


    // ========================================================================
    // CASE 3B
    //
    // Opposite side is empty.
    //
    // C is OUT according to this first implementation.
    //
    // D1 remains available for the search queue.
    // ========================================================================

    if (!hitB3.valid)
    {
        result.inside =
            0;

        return result;
    }


    // ========================================================================
    // CASE 3C
    //
    // Opposite side contains another center.
    //
    // It could be:
    //
    //      same D1
    //
    // or:
    //
    //      another D2
    //
    // If it is the same D1, we don't need to queue it twice.
    // ========================================================================

    if (
        !SameTrailCenter(
            hitB3.center,
            D1.center
        )
    )
    {
        result.D2 =
            MakeTrailCandidate(
                hitB3
            );
    }


    // Neither B2 nor B3 gave C.
    //
    // C is OUT according to this first/simple rule.
    result.inside =
        0;


    return result;
}


// ============================================================================
// CHECK WHETHER A CANDIDATE IS ALREADY IN OUR QUEUE
// ============================================================================

bool CandidateAlreadyExists(
    TrailCandidate candidates[TRAIL_MAX_CANDIDATES],
    uint candidateCount,
    TrailCandidate candidate)
{
    if (!candidate.valid)
    {
        return true;
    }


    [unroll]
    for (
        uint i = 0;
        i < TRAIL_MAX_CANDIDATES;
        ++i)
    {
        if (i >= candidateCount)
        {
            break;
        }


        if (
            SameTrailCenter(
                candidates[i].center,
                candidate.center
            )
        )
        {
            return true;
        }
    }


    return false;
}


// ============================================================================
// PUSH D1 / D2 INTO THE SEARCH QUEUE
// ============================================================================

void PushCandidate(
    inout TrailCandidate candidates[TRAIL_MAX_CANDIDATES],
    inout uint candidateCount,
    TrailCandidate candidate)
{
    if (!candidate.valid)
    {
        return;
    }


    if (
        candidateCount >=
        TRAIL_MAX_CANDIDATES
    )
    {
        return;
    }


    if (
        CandidateAlreadyExists(
            candidates,
            candidateCount,
            candidate
        )
    )
    {
        return;
    }


    candidates[candidateCount] =
        candidate;


    candidateCount++;
}


// ============================================================================
// FULL SEARCH
// ============================================================================

TrailSearchResult FindTrailImpulse(
    float3 worldPos)
{
    TrailSearchResult result =
        (TrailSearchResult)0;


    // ========================================================================
    // A
    //
    // Original point.
    // ========================================================================

    float3 A =
        worldPos;


    // ========================================================================
    // B
    //
    // Vertical projection of A onto Z = 0.
    //
    // BA = A - B
    // ========================================================================

    float3 B =
        float3(
            A.xy,
            0.0f
        );


    float3 BA =
        A - B;


    uint samplesUsed =
        0;


    // ========================================================================
    // SAMPLE B
    // ========================================================================

    TrailHit hitB =
        SampleTrailMapWorld(
            B
        );


    samplesUsed++;


    // Nothing at B.
    if (!hitB.valid)
    {
        result.state =
            TRAIL_RESULT_NONE;

        result.samplesUsed =
            samplesUsed;

        return result;
    }


    // ========================================================================
    // C
    //
    // Reconstructed from:
    //
    //      B + RG_direction * Blue_distance
    // ========================================================================

    TrailCandidate C =
        MakeTrailCandidate(
            hitB
        );


    // ------------------------------------------------------------------------
    // Named vectors for debugging / RenderDoc.
    // ------------------------------------------------------------------------

    float3 BC =
        C.center - B;

    float3 CB =
        B - C.center;

    float3 CA =
        A - C.center;


    // Suppress unused warnings if these are currently only debug variables.
    BA = BA;
    BC = BC;
    CB = CB;
    CA = CA;


    // ========================================================================
    // CANDIDATE QUEUE
    //
    // Initially:
    //
    //      candidates[0] = C
    //
    // Later:
    //
    //      candidates[1] = D1
    //      candidates[2] = D2
    //
    // Each candidate contains the point which discovered it, so the exact
    // same TestCandidate() function can be used recursively/iteratively.
    // ========================================================================

    TrailCandidate candidates[TRAIL_MAX_CANDIDATES];


    [unroll]
    for (
        uint i = 0;
        i < TRAIL_MAX_CANDIDATES;
        ++i)
    {
        candidates[i] =
            (TrailCandidate)0;
    }


    uint candidateCount =
        0;


    uint candidateIndex =
        0;


    PushCandidate(
        candidates,
        candidateCount,
        C
    );


    // ========================================================================
    // SEARCH LOOP
    // ========================================================================

    [loop]
    while (
        candidateIndex < candidateCount &&
        samplesUsed < TRAIL_MAX_SAMPLES)
    {
        TrailCandidate current =
            candidates[candidateIndex];


        // --------------------------------------------------------------------
        // Test:
        //
        //      C
        //      then potentially D1
        //      then potentially D2
        //
        // all using exactly the same routine.
        // --------------------------------------------------------------------

        CandidateTestResult test =
            TestCandidate(
                A,
                current,
                samplesUsed
            );


        // --------------------------------------------------------------------
        // Current candidate confirmed.
        // --------------------------------------------------------------------

        if (test.inside)
        {
            result.state =
                TRAIL_RESULT_FOUND;


            result.samplesUsed =
                samplesUsed;


            result.center =
                current.center;


            result.power =
                current.power;


            return result;
        }


        // --------------------------------------------------------------------
        // Candidate did not contain A according to this first implementation.
        //
        // But its B2/B3 samples may have revealed D1 and D2.
        // --------------------------------------------------------------------

        PushCandidate(
            candidates,
            candidateCount,
            test.D1
        );


        PushCandidate(
            candidates,
            candidateCount,
            test.D2
        );


        candidateIndex++;
    }


    // ========================================================================
    // FINISHED
    // ========================================================================

    result.samplesUsed =
        samplesUsed;


    if (
        samplesUsed >= TRAIL_MAX_SAMPLES &&
        candidateIndex < candidateCount
    )
    {
        result.state =
            TRAIL_RESULT_BUDGET_END;
    }
    else
    {
        result.state =
            TRAIL_RESULT_NONE;
    }


    return result;
}


// ============================================================================
// EXAMPLE VEGETATION VERTEX USAGE
// ============================================================================

void ApplyTrailImpulse(
    float3 worldPos,
    out float impulsePower,
    out float3 impulseCenter,
    out uint debugState,
    out uint debugSamples)
{
    TrailSearchResult trailResult =
        FindTrailImpulse(
            worldPos
        );


    debugState =
        trailResult.state;


    debugSamples =
        trailResult.samplesUsed;


    impulsePower =
        0.0f;


    impulseCenter =
        float3(
            0.0f,
            0.0f,
            0.0f
        );


    if (
        trailResult.state ==
        TRAIL_RESULT_FOUND
    )
    {
        impulsePower =
            trailResult.power;


        impulseCenter =
            trailResult.center;
    }
}










// ============================================================================
// TRAIL MAP 3D SEARCH - FIRST VERSION
// ============================================================================
//
// COORDINATE CONVENTION
//
// A = original 3D vegetation/pixel point (worldPos)
// B = vertical projection of A onto trail-map plane Z = 0
// C = first impulse center reconstructed by sampling trail map at B
//
// TrailMap:
//      R,G = normalized direction from sampled pixel -> impulse center
//      B   = normalized distance from sampled pixel -> impulse center
//      A   = impulse power / temporal decay
//
// FIRST IDEA:
//
//      A
//      |
//      |
//      B ---------------- Z = 0
//
// Sample B:
//
//      B -> C
//
// Calculate:
//
//      CA = A - C
//      distanceCA = length(CA)
//
//      CB = normalize(B - C) on the Z=0 plane
//
// First test point:
//
//      B2 = C + CB * distanceCA
//
// Sample B2:
//
//      B2 -> C
//          C is confirmed.
//          A is inside C.
//
//      B2 -> EMPTY
//          C is OUT using this first approximation.
//
//      B2 -> D1
//          D1 may simply be overwriting C.
//          Therefore check opposite side:
//
//          B3 = C - CB * distanceCA
//
//          B3 -> C
//              C confirmed.
//
//          B3 -> EMPTY
//              C out.
//              D1 is queued and tested.
//
//          B3 -> D1
//              C out.
//              D1 queued.
//
//          B3 -> D2
//              C out.
//              D1 and D2 queued.
//
// D1 / D2 use THE EXACT SAME TEST.
//
// Example









//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint uImpulses = 0;

    float2 vTotalDir = float2( 0.0f, 0.0f );
    float fMaxRadius = 0.0f;
    float fRetRadius = 0.0f;
    float fRetPower = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld = WORLD_PIVOT + ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir = vWorld - GET_IMPULSE_POS( j );
            float fSqrDist = 1.0f - saturate( dot( vDir, vDir ) * GET_IMPULSE_RCP_RADIUS( j ) );

            if( fSqrDist > 0.0f )
            {
                float fRadius = sqrt( 1.0f / GET_IMPULSE_RCP_RADIUS( j ) );

                vTotalDir += vDir * fSqrDist;
                uImpulses++;
                fMaxRadius = max( fMaxRadius, fRadius );
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev = texHistory.Sample( samplerHistory, i.vTex0 - UV_DISPLACEMENT );

    float fPrevPowerRaw = TrailMap_GetPower( vPrev );
    float fPrevPower = saturate( fPrevPowerRaw - DECAY );

    float fPrevRadius = 0.0f;

    if( fPrevPowerRaw > 0.0001f )
    {
        float fDecayRatio = fPrevPower / fPrevPowerRaw;
        fPrevRadius = vPrev.b * fDecayRatio;
    }


    // --------------------------------------------------------
    // OUTPUT
    // --------------------------------------------------------

    float2 vRetDir = float2( 0.5f, 0.5f );

    if( uImpulses > 0 )
    {
        // Original structure:
        // current impulse writes current result
        float2 vAvgDir = vTotalDir / uImpulses;
        float fCurrentRadius = saturate( fMaxRadius / MAX_IMPULSE_RADIUS );

        vRetDir = saturate( (vAvgDir / MAX_IMPULSE_RADIUS) * 0.5f + 0.5f );

        // If previous decayed radius is still bigger, keep the whole previous sphere
        // so a smaller current impulse cannot reactivate or cut it.
        if( fPrevPower > 0.0f && fPrevRadius > fCurrentRadius )
        {
            vRetDir = vPrev.rg;
            fRetRadius = fPrevRadius;
            fRetPower = fPrevPower;
        }
        else
        {
            fRetRadius = fCurrentRadius;
            fRetPower = 1.0f;
        }
    }
    else
    {
        // No current impulse: keep previous sphere, but both power and radius decay
        vRetDir = vPrev.rg;
        fRetRadius = fPrevRadius;
        fRetPower = fPrevPower;
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = packed non-normalized XY offset
    // B  = packed radius
    // A  = power / decay
    // --------------------------------------------------------

    float4 vRet = float4( 0.0f, 0.0f, 0.0f, 0.0f );

    vRet.rg = vRetDir;
    vRet.b = fRetRadius;
    vRet.a = fRetPower;

    return vRet;
}





test
//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint uImpulses = 0;

    float2 vTotalDir = float2( 0.0f, 0.0f );
    float fMaxRadius = 0.0f;
    float fRetRadius = 0.0f;
    float fRetPower = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld = WORLD_PIVOT + ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir = vWorld - GET_IMPULSE_POS( j );

            float fSqrDist = 1.0f - saturate( dot( vDir, vDir ) * GET_IMPULSE_RCP_RADIUS( j ) );

            if( fSqrDist > 0.0f )
            {
                float fRadius = sqrt( 1.0f / GET_IMPULSE_RCP_RADIUS( j ) );

                vTotalDir += vDir * fSqrDist;

                uImpulses++;

                fMaxRadius = max( fMaxRadius, fRadius );
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev = texHistory.Sample( samplerHistory, i.vTex0 - UV_DISPLACEMENT );

    float fPrevPower = saturate( TrailMap_GetPower( vPrev ) - DECAY );


    // --------------------------------------------------------
    // OUTPUT
    // --------------------------------------------------------

    float2 vRetDir = float2( 0.5f, 0.5f );

    if( uImpulses > 0 )
    {
        // Keep the original accumulation style, but RG is now
        // NON-normalized so its magnitude is preserved.
        float2 vAvgDir = vTotalDir / uImpulses;

        vRetDir = saturate( (vAvgDir / MAX_IMPULSE_RADIUS) * 0.5f + 0.5f );

        // Current maximum radius packed into B.
        float fCurrentRadius = saturate( fMaxRadius / MAX_IMPULSE_RADIUS );


        // ----------------------------------------------------
        // PREVIOUS BIGGER SPHERE WINS
        //
        // Keep RG + B + A together from history.
        // A smaller current impulse cannot refresh the
        // lifetime of an older larger sphere.
        // ----------------------------------------------------

        if( fPrevPower > 0.0f && vPrev.b > fCurrentRadius )
        {
            vRetDir = vPrev.rg;
            fRetRadius = vPrev.b;
            fRetPower = fPrevPower;
        }
        else
        {
            vRetDir = vRetDir;
            fRetRadius = fCurrentRadius;
            fRetPower = 1.0f;
        }
    }
    else
    {
        vRetDir = vPrev.rg;
        fRetRadius = vPrev.b;
        fRetPower = fPrevPower;
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = packed non-normalized XY offset
    // B  = packed impulse radius
    // A  = power / decay
    // --------------------------------------------------------

    float4 vRet = float4( 0.0f, 0.0f, 0.0f, 0.0f );

    vRet.rg = vRetDir;
    vRet.b = fRetRadius;
    vRet.a = fRetPower;

    return vRet;
}
